# ERS-01 — Engine Room Simulator

## What this is

ERS-01 is a browser-based engine room simulator for maritime engineering training. It reproduces the core systems of a ship's engine room — main engine, shaft line, generators, switchboard, and auxiliary machinery — as an interactive model that trainees and instructors can operate, observe, and troubleshoot in real time, entirely on a local machine.

It is built around STCW-style training: an instructor can run scenarios that introduce faults or abnormal conditions, watch how a trainee responds, and review the session afterward.

## What it does

- Simulates a diesel main engine with realistic behavior for starting air, clutch engagement, shaft RPM, load, cooling (HT/LT circuits), and lubrication.
- Models auxiliary machinery — fuel, cooling, and lube oil pumps, air compressors, purifiers, boiler, and generators — each with its own running state and possible failure modes.
- Drives an electrical switchboard with multiple generators, breaker logic, and blackout/blackout-recovery behavior.
- Raises alarms for abnormal conditions (overload, overspeed, low lube oil pressure, high cooling temperature, low starting air, misfire, purifier fire, MARPOL-related limits, and more).
- Runs predefined training scenarios (JSON-based) that inject faults on a timer or under specific conditions, for structured exercises.
- Scores trainee responses against exercise criteria and reports reaction times.
- Records every session to a log file, which can be replayed step-by-step afterward for debriefing.
- Supports English and Romanian interface languages.

## How it works

The system has two parts: a Python backend that runs the physics simulation, and a set of browser pages that display it and send commands.

1. `server.py` starts a local web server and a WebSocket server.
2. `engine_core.py` holds the actual simulation model — engine, generators, auxiliaries — and advances it one tick at a time.
3. The browser pages (`sala-masini.html` for the engine room, `tablou-electric.html` for the switchboard, `pid-schema.html` for the piping & instrumentation schematic, `configurare.html` for engine setup, `instructor.html` for the instructor console) connect over WebSocket, send operator commands (start/stop, throttle, mode selection, fault injection, etc.), and receive the live simulation state to display gauges, lamps, and animations.
4. `scenario_engine.py` can load a JSON scenario file and trigger scripted events — a fault, an alarm, a condition — at the right moment during a run.
5. `exercise_eval.py` checks the current state against exercise criteria to produce a score.
6. Every session is written to a log file that the instructor console can load back into a scrubber-style replay view, with a filterable event timeline and an exportable debrief report.

## Running it

```bash
python3 server.py
```

Then open in a browser:

```text
http://localhost:3000/sala-masini.html
```

Requires Python 3 (standard library only — the WebSocket server is implemented directly on `asyncio`, no extra packages needed).

## Project layout

```text
.
├── engine_core.py          # simulation model (engine, auxiliaries, generators, alarms)
├── server.py                # HTTP + WebSocket server, session logging
├── scenario_engine.py        # loads and runs JSON training scenarios
├── exercise_eval.py           # scores trainee responses against exercise criteria
├── coach_engine.py            # rule-based hints/feedback engine
├── excel_handler.py            # engine profile import/export via Excel
├── common.js                    # shared WebSocket client, translations, UI helpers
├── style.css                     # shared styling
├── sala-masini.html               # engine room control page
├── tablou-electric.html            # electrical switchboard page
├── pid-schema.html                  # piping & instrumentation diagram page
├── configurare.html                  # engine configuration page
├── instructor.html                    # instructor console, scenarios, replay studio
├── engines_config.json                 # engine model presets
├── scenarios/                           # training scenario definitions (JSON)
└── logs/                                 # recorded session logs
```

## Intended use

This is a training and demonstration platform, meant for local use in a classroom or self-study setting — not a certified, full-scale simulator equivalent to commercial systems such as Kongsberg or Wärtsilä. It is well suited to teaching operating procedures, fault recognition, and STCW-style competency assessment in a lightweight, self-contained environment.
