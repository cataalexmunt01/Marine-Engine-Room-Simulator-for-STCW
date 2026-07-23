# User Guide — Instructor

Your work interface: instructor.html. This guide describes exactly the controls in the application, functionally verified.

## 1\. New session

The "Start new session" button sends the new\_session action — clears the student's action log and resets the e-Coach, without restarting the physics engine. Use it between two different students on the same installation.

The "Reset completely" button sends the reset action — fully rebuilds the installation from scratch (engine, generators, electrical bus, all auxiliary units). Use it when you need a clean slate (e.g. after a catastrophic state during an exercise).

## 2\. Uploading an STCW exercise

The exercise panel loads the list from scenarios/exercise\_pack.json (35 exercises as of this release). By selecting an exercise, the app:

* Sends load\_scenario with the complete associated JSON scenario.

* Resets the e-Coach and loads the rules specific to that scenario (if any).

* Starts the internal evaluation timer.

At the end of the exercise (or upon request), the evaluation (evaluate\_exercise) runs — the result appears in a modal with score, STCW report by category, and timeline of the student's actions. The evaluation report labels respect the UI language (EN/RO).

## 3\. Live fault injection (Phase 4\)

This is the central panel for dynamic training: you can modify a plant state variable in real time using a signal shape of your choice. Each trigger is sent as an inject\_event action with a continuous\_trigger.

### 3.1 Available variables (functionally verified)

| Variable | What it represents |
| :---- | :---- |
| lube\_oil\_pressure | Lubrication oil pressure (bar) |
| cooling\_water\_temp | Cooling water temperature (°C) |
| air\_pressure | Starting air pressure (bar) |
| thrust\_bearing\_temp | Thrust bearing temperature (°C) |
| sewage\_tank\_level | Sewage tank level (%) |
| bilge\_level | Bilge level (%) |
| fuel\_service\_tank\_level | Fuel service tank level (%) |
| incinerator\_temp | Incinerator temperature (°C) |

### 3.2 Available signal shapes

| Shape | Parameters | Behaviour |
| :---- | :---- | :---- |
| Ramp (ramp) | start value, end value, duration | Linear interpolation from start to end over the specified duration |
| Step (step) | initial value, step value, delay | Remains at initial value until delay, then switches instantly |
| Single pulse (pulse) | base value, pulse value, pulse width | Pulse value for the duration of the width, then returns to base — once |
| Pulse train (pulse\_train) | base value, pulse value, period, pulse width | Repetitive pulses with the specified period |
| Sinusoidal (sine) | base value (opt.), amplitude, frequency (Hz) | Continuous sinusoidal oscillation around the base value |
| Random | base value, noise amplitude, "Random Walk" switch | Uniform noise around base; with Random Walk active, deviation accumulates progressively |

The general "duration" field sets the total trigger duration — upon expiration the variable returns to its base value (except for ramp, which remains at the final value).

### 3.3 Pedagogical good practices

* Use ramp for realistic progressive degradations (e.g. slow oil pressure drop) — useful for early failure recognition exercises.

* Use step for sudden, instantaneous failures (e.g. pump failure) — tests reaction time.

* Use sinusoidal/random to simulate instrument instability/noise without a real background fault — useful for testing whether the student can distinguish a normal fluctuation from a real fault.

* Combine with dedicated fault commands (section 4\) for compound scenarios (e.g. ramp on thrust\_bearing\_temp \+ simultaneous lubrication fault).

## 4\. Dedicated fault controls (beyond the trigger panel)

Some Phase 1 failures are injected via dedicated scenario actions — you can trigger them from the scenario panel or by editing a scenario file:

| Action | Effect |
| :---- | :---- |
| set\_misfire (with cylinder: 0-5) | Triggers misfire on the indicated cylinder |
| set\_thrust\_bearing\_fault | Activates thrust bearing failure |
| set\_turbo\_fault | Activates turbocharger surging predisposition (requires sudden load variation to manifest) |
| purifier\_fire\_start | Starts fire in the separator compartment |
| alarm\_raised / alarm\_cleared (with key) | Manually force/clear any alarm from the alarm panel |

## 5\. Dashboard — monitoring

The instructor panel displays in real time: all active alarms, the status of each auxiliary unit, current e-Coach messages, and the student's action log. Use the action log to verify the actual order of the student's interventions, not just the final result.

## 6\. STCW assessment and history

* STCW category report: each exercise criterion is labelled with an STCW competency; the report shows the percentage achieved in each category separately, not just the overall score. Labels are shown in the UI language (EN/RO).

* Score history: the results of previous exercises in the session remain available for comparison.

* Filterable log: the action timeline can be filtered for targeted review (e.g. only actions around a specific trigger).

* Session log export: the "Download CSV" button exports the full session event log (one row per second) for external analysis or replay.

## 7\. Session and scenario reset — three distinct actions

| Action | Button in instructor.html | Effect |
| :---- | :---- | :---- |
| new\_session | "Start new session" | Clears action log \+ resets e-Coach. Physics engine and scenario continue. Use between students. |
| reset | "Reset completely" | Fully rebuilds engine, generators, bus and auxiliaries from scratch. Also clears log. |
| reset\_scenario | "Reload current scenario" | Reloads the last loaded exercise/scenario (falls back to scenario\_demo.json only if none was loaded yet this session). Accessible from the UI via the "Reload current scenario" button. |

## 8\. P\&ID page (pid-schema.html)

The P\&ID page is accessible from the navigation bar on all pages. It shows the full plant piping diagram with live gauges.

## 9\. Engine profile configuration (configurare.html)

The configuration page allows selecting the main engine and generator profiles for the session. It also supports importing new engine profiles via an Excel template (Template\_Profiluri\_Motoare\_ERS01.xlsx). To add a custom engine profile:

* Download the Excel template from the configuration page.

* Fill in the engine parameters following the column headers.

* Upload the completed file — the server parses it and merges new profiles into engines\_config.json.

## 10\. Language

The interface defaults to English. The language selector in the navigation bar switches between English and Romanian. e-Coach messages and exercise titles/objectives/descriptions follow this setting as well.