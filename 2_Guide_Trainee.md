# User Guide — Trainee (Engine Room)

Your work interface: sala-masini.html. This guide describes exactly the controls and behaviours existing in the application.

## 1\. Starting the session

When the page loads, the simulator displays the initial state (engine off, air pressure full, basic systems on). The instructor can load an exercise scenario — if they have done so, you will see the objectives of the current exercise.

The application has five pages that you will use as a learner:

* sala-masini.html — the main panel: main engine and auxiliary systems (the subject of this guide).

* tablou-electric.html — electrical panel: generator start-up, synchronisation and circuit breaker closing for busbar paralleling.

* configurare.html — engine/generator profile selection for the session (usually prepared by the instructor before the exercise).

* pid-schema.html — interactive P\&ID diagram showing the full plant piping, with live instrument gauges and quick-access equipment controls.

* instructor.html — instructor console (not a trainee page, but accessible in the same browser).

### 1.1 Electrical panel — starting the generators

For each generator you have four commands: Start, Stop, Synchro \+ Circuit Breaker, Open Circuit Breaker.

* Start starts the generator; it takes approximately 8 seconds to reach rated speed.

* Sync \+ Circuit Breaker closes the circuit breaker — it only works if the generator is already at rated speed (within ±3% tolerance). If you press it too early, a desynchronisation alarm occurs, just like a real wrong paralleling.

* Without at least one generator running and the circuit breaker closed, the electrical bus is in blackout — all main engine electric pumps (cooling, lubrication) fail, regardless of their local switch.

## 2\. Starting the main engine

Press Start. The main engine is powered by compressed air from cylinders, just like on a real ship:

* If the air pressure is below 18 bar, the command is rejected — the engine will not start no matter how many times you press Start. You must first start an air compressor (section 5\) and wait for the pressure to build up.

* A successful start consumes 6 bar of air reserve. The pressure is only restored by the compressor, never by itself.

* The engine has a starting ramp of approximately 15 seconds until it reaches a useful idle — you will not be at full speed right away.

Press Stop to turn off the engine. Stopping automatically disengages the clutch, for safety.

## 3\. Local lever vs. telegraph order

* The local lever (the slider on screen) is what actually controls the engine (0 \= STOP, full forward/reverse).

* Any telegraph order from the bridge is informative only — you, as the engineer, must move the local lever to match it. The engine does not move by itself.

## 4\. Operating mode (PTO / MEC / PTI)

Three buttons: PTO, MEC, PTI.

* MEC — the main engine drives the propeller alone; diesel generators cover the ship's own consumption.

* PTO — the main engine drives the propeller AND, through the shaft generator, produces additional electrical current.

* PTI — the main engine is stopped; the shaft is electrically driven by generators (electric motor mode).

Only change the mode when you are certain of the installation's condition — the change immediately affects load distribution.

## 5\. Auxiliary systems

The right panel lists all pumps, compressors and separators, grouped by category: seawater, freshwater, lubricating oil, fuel, starting air, separators, bilge/sewage/incinerator.

Each unit has three possible states:

* OFF — ready to be started manually.

* ON — working.

* FAILURE — defective; cannot be started until reset/repaired.

Each backup unit (e.g. large water pump 2, air compressor 2\) remains off by default — you must turn it on if the main unit fails. There is no automatic redundancy.

## 6\. Clutch engagement (propeller)

The Engage / Disengage buttons control the coupling between the engine and propeller shaft — a separate operation from starting/stopping the engine.

* Engagement is only successful if the engine is running AND its speed is within a narrow idle band (displayed on screen). Beyond this band the command is rejected — attempting to engage at high speed would cause mechanical shock to the gearbox.

* Engagement is not instantaneous — it takes approximately 4 seconds for the clutch to fully lock.

* If you remain engaged with high slip for more than 15 seconds, the excessive slip alarm appears — check load and speed.

## 7\. Alarms

The alarm panel on the right automatically lights up when a real condition is exceeded — they are not decorative. Main categories:

| Alarm | Typical cause |
| :---- | :---- |
| ME Overload / Excessive Speed | Requested load above capacity |
| Low oil pressure | Oil pump stopped/defective, or blackout |
| High cooling temp. | Insufficient cooling pumps (sea/fresh water) |
| Low starting air | Air reserve below threshold — start the compressor |
| Cylinder misfire | One cylinder burns incompletely — reduce load |
| Support bearing — high temp. | High load on the shaft or lack of lubrication |
| Surging turbocharger | Sudden load variation combined with turbo fault |
| Fire — separators | See section 8 — immediate response required |
| High sewage/bilge level | Start the corresponding pump |
| Low service tank fuel | Start the transfer pump |
| Incinerator — over-temperature | Check combustion regime |
| High fuel viscosity | Fuel heater not running or boiler pressure low |
| Boiler low water level | Boiler feed pump stopped or bus in blackout |
| Emissions violation | SOx or NOx above MARPOL limits — check scrubber/SCR |
| MARPOL violation | Environmental alarm active for \>30 s without reaction |

## 8\. Fire in the separator enclosure — response procedure

If this fire occurs, a dedicated red panel appears on screen with two buttons:

* Close the fuel QCV (quick-closing valve).

* Release CO₂ — carbon dioxide flooding.

Both actions are required — one alone will not extinguish the fire. The order does not matter, but if you abandon one of them partway through, the extinguishing timer resets. After extinguishing, the fuel separator remains in a fault (fire-damaged) state and must be manually reset from the auxiliary systems panel before it can be restarted.

## 

## 9\. P\&ID diagram (pid-schema.html)

The P\&ID page shows the full plant piping in an interactive diagram. Live instrument gauges update in real time. You can click on any equipment node to open a modal with Start / Stop / Inject fault / Repair buttons.

## 10\. e-Coach — support messages

During the exercise, an assistant (e-Coach) can send you contextual messages: suggestions (yellow), critical warnings (red), or success confirmations (green). Messages appear only when a condition remains true for several seconds (not for short transients) and do not repeat immediately. You can acknowledge/close a read message.

## 11\. Exercises and scoring

If the instructor has loaded an STCW exercise, at the end you will receive a report with:

* Overall score (0–100) — percentage of possible points obtained.

* "Completed" status — requires a score ≥ 70 AND no penalty applied (a single dangerous action can invalidate the exercise even with an otherwise good raw score).

* STCW category report — how much you achieved in each assessed competency.

* Action timeline — a list of your actions, useful for review with your instructor.

## 12\. Common mistakes to avoid

* Do not wait for backup systems to start by themselves — they must be commanded manually.

* Do not ignore a "minor" alarm for long — many evolve into penalties if you do not react (e.g. sewage/bilge, low fuel, MARPOL).

* Check idle speed before engaging the clutch — the rejected command is not a bug, it is the safety interlock.

* On a separator fire, do not stop at just one action — you need both (QCV close AND CO₂ release).

* Keep at least one generator running and its circuit breaker closed before starting engine electric pumps; a blackout disables them all.