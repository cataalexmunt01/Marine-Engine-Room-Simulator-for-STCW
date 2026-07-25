# ERS-01 Engine Room Simulator — Instructor Manual

### Exercise Catalogue: Theory, Procedure, and STCW-Aligned Assessment

---

## About this manual

ERS-01 is a browser-based engine room simulator used to run structured, STCW-style training exercises: the instructor loads a scenario, the simulator injects a fault or abnormal condition at a scripted moment, the trainee responds at the control stand, and the system scores the response against fixed criteria. Every session is logged and can be replayed for debrief.

This manual documents all 33 exercises currently defined in the scenario pack (`scenarios/exercise_pack.json`). For each exercise it gives:

- **What is trained** — the underlying engineering theory and why it matters operationally.
- **What the instructor does** — exactly which fault the scenario engine injects, and when.
- **What the trainee is expected to do** — the correct operational response.
- **How the instructor/system checks and scores it** — the exact criteria, points, and (where applicable) time limits used by the automatic evaluator.
- **STCW category** — the competence area the exercise is tagged against.
- **Debrief notes** — common trainee errors and points worth raising during the replay/debrief.

Exercises are grouped into five parts for teaching progression:

1. Main Engine Fault Recognition & Protection
2. Electrical Power & Blackout Management
3. Emergency & Casualty Control
4. Auxiliary Systems & Steam Plant
5. Environmental Compliance (MARPOL)

A quick-reference table of all 33 exercises is provided in the Appendix.

### How the automatic scoring works

The evaluator (`exercise_eval.py`) checks three kinds of criteria against the trainee's action log and the simulator's final state:

| Criterion type | What it checks | Typical use |
|---|---|---|
| `state` | Whether a variable (e.g. `running`, `cooling_water_temp`, `air_pressure`) satisfies a condition (`==`, `<`, `>`, `<=`, `>=`, `!=`) at the point of evaluation | "Is the engine stopped?" / "Is temperature back under 85 °C?" |
| `sequence` | Whether a required chain of actions occurred in order (not necessarily consecutively) | "Did the trainee start the standby pump *before* clearing the alarm?" |
| `reaction_time` | The time between a trigger event (e.g. an alarm) and the correct response action, checked against a maximum allowed time | "Did the trainee stop the engine within 20 s of the low lube-oil alarm?" |
| `penalty` | A negative-points rule that fires when the trainee does something unsafe (e.g. attempts a start without pre-lubrication, closes a breaker out of synchronism) | Deducted from the total regardless of whether the main objective was met |

Each criterion carries a `stcw` tag. Scores are aggregated **per STCW category** as well as overall, so a trainee's report shows not just a final percentage but a breakdown by competence area — useful for identifying a specific gap (e.g. consistently strong on machinery protection but weak on electrical safety) across a training course rather than a single exercise.

All exercises in this pack are normalized to **100 maximum positive points**; penalties are subtracted from whatever positive score was earned.

### Using this manual with the simulator

For each session: load the scenario from the instructor console (`instructor.html`), brief the trainee only on the general watchkeeping context (not the fault itself — that defeats the purpose), run the scenario, and use the automatic scorecard plus the replay/timeline feature for a structured debrief immediately afterward. The theory sections below are intended as instructor talking points during that debrief, not as material to hand to the trainee in advance.

---

## Part 1 — Main Engine Fault Recognition & Protection

These exercises train the core operational-level competence of *operating main and auxiliary machinery and associated control systems*, with an emphasis on recognizing an abnormal parameter and acting before it becomes a mechanical failure.

---

### 1. Demo: Fuel Supply Failure

**Scenario ID:** `demo-fuel-failure` · **STCW category:** ERM / Troubleshooting

**Objective:** Identify and manage a fuel-supply failure on the main engine — this is the introductory exercise for the whole course.

**Theory:** A diesel main engine depends on a continuous, pressurized fuel supply from the booster/circulating pump. Loss of that supply causes fuel rack starvation, irregular combustion, and eventually stalling under load. Most installations carry a standby fuel pump specifically for this reason. The correct response is to recognize the pressure/flow anomaly quickly, isolate the failed unit, and either bring the engine to a safe, protected stop or restore supply before consequential damage (misfiring, thermal shock, loss of propulsion) occurs.

**What the instructor/scenario does:** The scenario engine fails the fuel auxiliary pump (`aux_fail: fuel`) 5 seconds into the run, then fails the fresh-water cooling pump (`aux_fail: fw1`) at 12 seconds — layering a second fault so the trainee must prioritize.

**Expected trainee response:**
- Recognize the fuel-pressure/alarm indication.
- Stop the failed fuel pump and either bring the standby pump on line or bring the engine to a controlled, protected stop.
- Do not ignore the second (cooling) fault while resolving the first.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Main engine secured (stopped) or safely recovered | `state` (`running == false`) | 50 | Final state check |
| Procedure: stop failed fuel pump, then stop engine | `sequence` (`aux_stop:fuel → stop`) | 50 | Order matters — stopping the pump before the engine reflects correct fault isolation |

**Debrief notes:** This is the exercise where instructors should establish the baseline habit: *isolate the fault, then decide on continuation vs. shutdown* — not the reverse. Watch for trainees who stop the engine reflexively without acknowledging the fuel pump fault at all (they will pass the state check but miss the sequence check, which is the actual teaching point).

---

### 2. Low Lubricating Oil Pressure

**Scenario ID:** `low-lube-oil-pressure` · **STCW category:** STCW III/1 — Machinery Protection

**Objective:** React to a critical drop in lubricating-oil pressure and protect the main engine.

**Theory:** Lubricating oil pressure maintains the hydrodynamic film at main and crankpin bearings, camshaft bearings, and piston cooling galleries. A sustained pressure drop below the safe threshold leads to metal-to-metal contact within seconds under load — bearing wipe-out, scoring of journals, and in the worst case a seized crankshaft. This is why low LO pressure is one of the few conditions with an automatic engine shutdown interlock on most installations, and why manual reaction time is trained explicitly here.

**What the instructor/scenario does:** At 4 seconds, the lube-oil pump is failed (`aux_stop: lube_oil`) and lube-oil pressure is forced down to 1.0 (bar). If pressure remains below 1.5 bar, the simulator itself will auto-stop the engine as a protective interlock.

**Expected trainee response:**
- Recognize the low lube-oil-pressure alarm immediately.
- Stop the main engine within the allowed reaction window — do not wait for the automatic interlock.
- Never attempt to restart without the lube-oil pump confirmed running and pressure established (pre-lubrication).

**Assessment criteria (100 pts total, plus a penalty rule):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Main engine stopped for lubrication protection | `state` (`running == false`) | 40 | Final state |
| Rapid stop on low LO-pressure alarm | `reaction_time` (trigger: alarm → `stop`) | 60 | Max **20 s** |
| Penalty: attempted start without active pre-lubrication | `penalty` | −30 | Fires if `start` occurs while the LO pump was not confirmed running beforehand |

**Debrief notes:** The 20-second window is intentionally tight — it mirrors realistic bearing-damage timescales, not an arbitrary training limit. If a trainee stops the engine but takes 35 seconds, use the replay timeline to show exactly how much of that was hesitation versus correct-but-slow execution. The penalty rule is where trainees who get complacent about pre-lubrication habits get caught, even in unrelated future exercises where they restart the engine out of habit.

---

### 3. High Cooling Water Temperature

**Scenario ID:** `high-cooling-water-temp` · **STCW category:** ERM / Technical Response

**Objective:** Isolate the cause of rising cooling-water temperature and recover stable cooling.

**Theory:** Jacket cooling water (HT circuit) removes combustion heat from cylinder liners and cylinder heads. A rising trend usually points to a seawater (LT/SW side) pump failure, a fouled cooler, or a closed valve rather than the HT circuit itself. Sustained operation above the safe limit (commonly cited around 85–90 °C for jacket water) accelerates liner wear, promotes micro-seizure risk from reduced piston-to-liner clearance, and can crack cylinder heads under thermal cycling. The correct fix is almost always to restore seawater cooling flow via the standby pump, not just to reduce load.

**What the instructor/scenario does:** No explicit event is scripted to force the fault in this particular file beyond the initial abnormal trend (the simulator's physics model drives the temperature rise from the reduced-flow condition set at scenario start); the trainee is scored on bringing temperature down and engaging the correct standby equipment.

**Expected trainee response:**
- Recognize the abnormal cooling-water temperature trend.
- Start the standby seawater pump (SW2).
- Bring temperature back to a normal operating value without running for an extended period at high temperature.

**Assessment criteria (100 pts total, plus a penalty rule):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Cooling water temperature restored to ≤ 85 °C | `state` | 40 | Final state |
| Standby seawater pump (SW2) engaged | `sequence` (`aux_start:sw2`) | 60 | Confirms correct corrective action, not just a lucky recovery |
| Penalty: prolonged running at critical temperature | `penalty` (`high_temp_run`) | −40 | Fires if the high-temperature alarm condition persists more than 20 s of cumulative duration |

**Debrief notes:** A trainee can sometimes "pass" the temperature check without ever touching SW2 if the model recovers on its own — the `sequence` criterion is there specifically to prevent instructors from crediting a lucky outcome instead of the correct action. Use this to reinforce diagnose-then-act discipline rather than wait-and-see.

---

### 4. Insufficient Starting Air

**Scenario ID:** `low-start-air` · **STCW category:** STCW — Starting Procedures & Safety

**Objective:** Understand the impact of reduced starting-air pressure on the start sequence.

**Theory:** Starting air (typically stored around 25–30 bar in the starting-air receivers) provides the torque to turn the engine over through the firing sequence. Below a minimum threshold, there is not enough energy to carry the crankshaft through compression on all cylinders, risking a failed or incomplete start, and on some engines the control system will simply block the start command via an air-pressure interlock. The correct response is always to build air pressure back up via a compressor before attempting to start — never to force a start on marginal air.

**What the instructor/scenario does:** At 2 seconds, air pressure is driven down to 16 bar and the running air compressor (AC1) is stopped (`aux_stop: ac1`), simulating a compressor trip that left the receiver undercharged.

**Expected trainee response:**
- Recognize the low air-pressure indication before attempting a start.
- Start compressor AC1 (or the available standby) to recharge the receiver.
- Only issue the main engine start command once pressure has recovered to a safe value.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Procedure: start AC1, then start main engine only after pressure recovers | `sequence` (`aux_start:ac1 → start`) | 100 | Order-dependent — starting the engine before the compressor fails this check even if the engine eventually starts |

**Debrief notes:** This is a pure sequencing exercise — full marks require patience. Trainees who try to "just start it anyway" because the simulator doesn't visibly resist should be shown, in the debrief, what a failed/incomplete start does to starting-air reserves and to the pistons/liners in a real engine (multiple failed start attempts can exhaust the entire receiver, leaving the vessel without means to restart at all).

---

### 5. Fuel Pump Failure (Under Load)

**Scenario ID:** `fuel-pump-failure` · **STCW category:** ERM / Supply Continuity

**Objective:** Switch to standby and prevent the main engine from stopping under load.

**Theory:** Unlike the introductory demo exercise, this drill is scored on *continuity of operation* rather than protective shutdown — reflecting the real operational priority when a vessel is underway and stopping the main engine is itself a hazard (loss of steerage way, proximity to other traffic, restricted waters). The correct response to a fuel pump failure while the engine must keep running is fast changeover to the standby pump, not a stop.

**What the instructor/scenario does:** At 4 seconds, the fuel pump fails (`aux_fail: fuel`) and throttle is forced down to 0.3, simulating the load instability that follows.

**Expected trainee response:**
- Recognize the fuel-supply fault without reflexively stopping the engine.
- Start the standby/auxiliary fuel pump promptly.
- Keep the main engine running and stable.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Main engine kept running | `state` (`running == true`) | 40 | Final state — this is the opposite polarity from most other drills, worth flagging to trainees |
| Standby fuel pump activated | `sequence` (`aux_start:fuel`) | 60 | Confirms deliberate corrective action |

**Debrief notes:** Trainees who have just completed Exercises 1–2 (where stopping the engine was the correct answer) sometimes over-generalize "when in doubt, stop" and lose points here. This is a deliberately placed contrast exercise — use it to teach that the correct response depends on load-bearing context (is stopping itself hazardous right now?), not on a single universal rule.

---

### 6. Crankcase Oil Mist / Explosion Hazard

**Scenario ID:** `oil-mist-hazard` · **STCW category:** STCW — Mechanical Emergency

**Objective:** React to a critical lubrication casualty and isolate the main engine before an explosion risk develops.

**Theory:** A hot spot inside the crankcase (from a wiped bearing or scored liner) vaporizes lubricating oil into a fine mist; if that mist reaches the right concentration and finds an ignition source, a crankcase explosion can follow, propagating destructively through the engine. This is precisely why oil-mist detectors are a mandatory fitment on medium/large marine diesel engines and why their alarm demands an *immediate* stop — there is no safe "monitor and see" option here, unlike a slow temperature drift. Crucially, once stopped, crankcase doors must **not** be opened immediately — trapped hot mist can flash on contact with fresh air, so doors are left closed until the crankcase has had time to cool.

**What the instructor/scenario does:** At 5 seconds, lube-oil pressure is dropped to 0.8 bar and cooling-water temperature is spiked to 104 °C simultaneously — a compound signature consistent with a serious internal lubrication casualty. If lube-oil pressure remains under 1.0 bar, the simulator auto-stops the engine.

**Expected trainee response:**
- Recognize the combined signature (pressure loss + temperature spike) as a serious casualty, not a routine alarm.
- Stop the main engine immediately — do not attempt any diagnostic troubleshooting while running.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Engine stopped in emergency to prevent crankcase explosion | `state` (`running == false`) | 100 | Full weight on a single, unambiguous action |

**Debrief notes:** Because this exercise carries its entire score on one action, it is a good place in the course to talk explicitly about the "no troubleshooting while running" rule for crankcase-related alarms, and to reference the standard post-incident cooling/ventilation wait before any crankcase door is opened — even though the simulator itself does not model door-opening.

---

### 7. Scavenge Fire

**Scenario ID:** `scavenge-fire` · **STCW category:** STCW — Internal Fire

**Objective:** Identify and limit a scavenge-space fire without aggravating the main engine casualty.

**Theory:** In two-stroke engines, oil and carbon deposits can accumulate in the scavenge air spaces; if they ignite (often from a blow-by or overheating piston ring), the resulting scavenge fire raises local temperatures sharply and can spread to adjacent structure. The universal first response is to reduce load/RPM immediately (which cuts scavenge air flow and starves the fire) while preparing or activating the fixed extinguishing arrangement (steam smothering or CO2 depending on the installation) — and critically, scavenge doors are **never** opened while the fire is suspected active, as the inrush of air will intensify it.

**What the instructor/scenario does:** At 4 seconds, cooling-water temperature is spiked to 103 °C and throttle is simultaneously forced down to 0.2, representing the combined thermal signature and the load reduction that must follow.

**Expected trainee response:**
- Recognize the abnormal temperature spike as a scavenge-fire indication rather than a routine cooling fault.
- Reduce load and stop the engine quickly, engaging fixed fire-fighting measures per procedure.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Engine stopped rapidly and fire-fighting system engaged | `state` (`running == false`) | 100 | Full weight on rapid, decisive shutdown |

**Debrief notes:** Contrast this with Exercise 3 (high cooling temperature), where the correct answer is to restore cooling and keep running — here the same symptom (rising temperature) has a different underlying cause and demands the opposite response (stop, don't just add cooling). This pairing is useful for teaching diagnostic reasoning rather than symptom-matching.

---

### 8. Cylinder Misfire (Incomplete Combustion)

**Scenario ID:** `misfire-cylinder` · **STCW category:** STCW — Marine Diesel Engines / Fault Diagnosis

**Objective:** Detect the misfiring cylinder from exhaust-gas temperature deviation and reduce engine load.

**Theory:** Each cylinder's exhaust gas temperature is monitored individually precisely so that a misfire (from a stuck/worn fuel injector, a leaking exhaust valve, or fuel-valve timing fault) shows up as a clear deviation from the other cylinders before it causes secondary damage (unburned fuel washing down the liner, carbon build-up, or turbocharger imbalance). The trained response is to identify the affected cylinder from the exhaust temperature spread and reduce overall engine load to limit stress while the fault is investigated — not to keep running at full load hoping it self-corrects.

**What the instructor/scenario does:** At 15 seconds into the run, cylinder 3 is set into a misfire condition (`set_misfire: cylinder 3`).

**Expected trainee response:**
- Monitor exhaust gas temperature spread across cylinders and identify the deviating cylinder.
- Reduce throttle/load promptly to limit further stress on the affected cylinder and its neighbors.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Engine load reduced after misfire detection | `state` (`throttle <= 0.4`) | 100 | Final throttle position |

**Debrief notes:** During replay, walk the trainee through the exhaust-gas-temperature trend for cylinder 3 versus the others so they connect the *pattern* (not just an alarm lamp) to the diagnosis — this is a core watchkeeping skill that instrumentation alone does not teach.

---

### 9. Thrust Bearing Overheating

**Scenario ID:** `thrust-bearing-overheat` · **STCW category:** STCW — Transmission Systems / Bearings

**Objective:** React to abnormal thrust-bearing temperature rise before the critical threshold is reached.

**Theory:** The thrust bearing (commonly a Michell/Kingsbury tilting-pad design) absorbs the axial (propeller) thrust and transmits it to the ship's hull. Overheating — from lubrication failure, misalignment, or pad damage — can progress to pad wipe-out and, in the worst case, loss of axial location of the shaft. Because thrust load scales with propeller thrust, the direct and fastest way to reduce bearing stress is to reduce shaft RPM/load; this is the correct immediate action while the underlying lubrication or alignment fault is investigated.

**What the instructor/scenario does:** At 10 seconds, a thrust-bearing fault is set (`set_thrust_bearing_fault: true`), driving abnormal temperature rise at the bearing.

**Expected trainee response:**
- Recognize the thrust-bearing temperature alarm.
- Reduce engine RPM/throttle promptly to relieve axial load on the bearing.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Throttle reduced to protect the thrust bearing | `state` (`throttle <= 0.3`) | 100 | Final throttle position |

**Debrief notes:** This exercise pairs well with the low-lube-oil drill (Exercise 2) to reinforce that not every lubrication-related casualty warrants a full stop — sometimes load reduction is the correct first-line response while a stop is held in reserve if temperature continues to climb.

---

### 10. Turbocharger Surge

**Scenario ID:** `turbocharger-surge` · **STCW category:** STCW — Turbochargers / Supercharging Systems

**Objective:** Recognize turbocharger surge and stabilize the engine by rapidly reducing load.

**Theory:** Surge occurs when the compressor side of the turbocharger cannot sustain steady forward airflow against the downstream pressure — commonly triggered by a rapid load/throttle increase that outpaces the turbocharger's ability to spool up, especially if there is an underlying fault (fouled compressor, damaged nozzle ring). The result is a rapid, audible flow reversal/pulsation that can damage compressor blades and disrupt combustion air supply. The only effective immediate response is to cut load quickly, which removes the exhaust energy driving the imbalance and lets the flow re-stabilize.

**What the instructor/scenario does:** A turbocharger fault is armed at 8 seconds (`set_turbo_fault: true`), then at 20 seconds the scenario forces a rapid load increase (`set_throttle: 0.95`) — deliberately recreating the classic trigger condition for surge.

**Expected trainee response:**
- Recognize the surge condition as load/throttle rises.
- Reduce throttle sharply to stabilize turbocharger airflow.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Load reduced to stop the surge | `state` (`throttle <= 0.3`) | 100 | Final throttle position |

**Debrief notes:** Use the replay to show the exact throttle trajectory the trainee commanded at 20 s — this exercise specifically tests whether the trainee reacts to a *rate of change* in commanded load, not just an absolute alarm threshold, which is a subtler skill than most of the other Part 1 drills.


---

## Part 2 — Electrical Power & Blackout Management

These exercises train the operational-level competence of *operating electrical, electronic, and control equipment*, with particular focus on blackout recovery sequencing and safe synchronization — areas where incorrect actions (e.g. closing a breaker out of synchronism) are penalized as sharply as failing to act at all.

---

### 11. Blackout and Recovery

**Scenario ID:** `blackout-recovery` · **STCW category:** STCW — Blackout Recovery

**Objective:** Recover electrical supply and operational stability after a loss of power.

**Theory:** A blackout (total loss of electrical power) removes all pump and control support from the main engine and auxiliaries simultaneously — it is one of the highest-consequence casualties in the engine room because everything downstream of the switchboard fails at once. Recovery follows a strict priority order: start the emergency/standby generator, bring it up to speed and voltage/frequency, synchronize it to the dead bus (or close directly onto a de-energized bus per the vessel's procedure), then restore auxiliaries in priority order (lube oil, cooling, fuel) before attempting to bring the main engine back to a running condition. Closing a generator breaker onto a live bus without synchronizing is itself a serious fault that can re-trip the new generator or cause equipment damage — hence it is explicitly penalized in this drill.

**What the instructor/scenario does:** At 3 seconds, the fuel auxiliary pump is failed (`aux_fail: fuel`) concurrent with the blackout condition, removing a supporting auxiliary at the same time as electrical power.

**Expected trainee response:**
- Recognize the blackout and initiate the standby generator start sequence without delay.
- Synchronize before closing the breaker — never close onto an unsynchronized bus.
- Restore the failed fuel pump once power is available.

**Assessment criteria (100 pts total, plus a penalty rule):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Blackout recovered (bus re-energized) | `state` (`bus.blackout == false`) | 40 | Final state |
| Standby generator started and breaker closed within time | `reaction_time` (trigger: `blackout_start` → `gen_close_breaker:1`) | 60 | Max **45 s** |
| Penalty: breaker closed without prior synchronization | `penalty` (`breaker_unsynced`) | −40 | Fires on an unsynchronized closure attempt or resulting re-trip |

**Debrief notes:** The 45-second window reflects a realistic, achievable target for a trained operator, not a "rush it" incentive — trainees who close the breaker fast but unsynchronized will lose more (−40) than they gain from speed. This is the single best exercise in the pack for teaching that *correct sequence beats raw speed*.

---

### 12. Total Blackout and Recovery

**Scenario ID:** `total-blackout-recovery` · **STCW category:** STCW — Emergency Recovery

**Objective:** Recover critical systems after total loss of power, at a more advanced/compounded level than Exercise 11.

**Theory:** This is the "dead ship" scenario taken further: not only electrical power but also starting air and key auxiliaries are lost simultaneously, closely resembling a real total-blackout casualty where cascading failures (a tripped generator, followed by loss of air-driven or electrically-driven support systems) leave the vessel without propulsion or steering power. The correct approach mirrors Exercise 11 but under compounded pressure: restore what can be restored first (typically air, since compressors themselves may need electrical power — prioritization judgment is part of the test), then rebuild toward a stable electrical and mechanical baseline methodically rather than trying to restart everything at once.

**What the instructor/scenario does:** At 2.5 seconds, the fuel pump is failed, the air compressor (AC1) is stopped, and air pressure is driven down to 8 bar — a compounded triple fault. A conditional recovery event resets the fuel pump and reduces throttle once air pressure drops under 10 bar, reflecting the knock-on effects the trainee must manage.

**Expected trainee response:**
- Diagnose the compounded nature of the fault (electrical + pneumatic + fuel) rather than treating it as a single-system casualty.
- Restore the electrical bus, then rebuild air and fuel support in a controlled sequence.
- Avoid attempting to restart the main engine before all supporting systems are confirmed available.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Emergency supply and main bus re-energized | `state` (`bus.blackout == false`) | 100 | Final state |

**Debrief notes:** Although the scoring is a single state check, the value of this exercise is entirely in the replay — walk through the order in which the trainee actually restored each system and compare it against the textbook dead-ship recovery priority list. A trainee can technically "pass" by restoring the bus while leaving auxiliaries in a poor state; use the debrief, not just the score, to catch that.

---

### 13. Generator Trip and Load Transfer

**Scenario ID:** `generator-trip` · **STCW category:** ERM / Electrical Generation

**Objective:** Identify the generator disturbance and manage load transfer to the standby generator.

**Theory:** When a running generator trips, the remaining generator(s) on the bus must absorb the lost load instantly, which can itself trigger an overload trip if the standby unit is not brought on line and synchronized quickly. This exercise also links electrical and pneumatic systems: a starting-air pressure drop is scripted alongside the trip, since air-driven auxiliaries (compressors, some pump drives) commonly share the same electrical dependency chain, and instructors use this pairing to test whether trainees can hold situational awareness across two systems at once rather than fixating on the more visible electrical alarm.

**What the instructor/scenario does:** At 3 seconds, the fuel pump is failed and air pressure is dropped to 16 bar, simulating the generator trip and its knock-on effects. A conditional event resets the fuel pump once air pressure falls under 18 bar, representing partial automatic recovery that the trainee must still confirm and stabilize.

**Expected trainee response:**
- Recognize the generator trip and the resulting bus disturbance.
- Start and synchronize generator 2 (or the designated standby) to take up the load without causing a blackout.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Generator 2 started and synchronized to take load | `state` (`bus.blackout == false`) | 100 | Final state — the bus staying live is the proxy for a correctly executed transfer |

**Debrief notes:** Because the scoring here is state-based rather than sequence-based, an instructor should use the action log/replay to confirm the trainee actually synchronized correctly rather than the bus simply never having fully collapsed. This is a good exercise for a verbal walk-through question: "at what point did you confirm synchronism before closing the breaker?"

---

### 14. Electrical Energy Management

**Scenario ID:** `energy-management` · **STCW category:** STCW III/1 — Energy Management

**Objective:** Manage load distribution and optimize consumption according to the vessel's operating mode (e.g. maneuvering in port versus free-running at sea).

**Theory:** Good electrical energy management is not just about avoiding blackouts — it is about running the minimum number of generators consistent with the load and its variability, since running unnecessary generators wastes fuel and increases maintenance load, while running too few risks an overload trip the moment load surges (as during maneuvering, when bow thrusters, mooring winches, and increased pump duty can spike demand sharply). The trained skill is anticipating load surges before they trip protection, and synchronizing/paralleling an additional generator proactively rather than reactively.

**What the instructor/scenario does:** At 5 seconds, an overload alarm on the main engine/bus is raised (`alarm_raised: overload_me`), representing a sudden load surge consistent with a maneuvering-type demand spike.

**Expected trainee response:**
- Anticipate or promptly recognize the load surge.
- Bring an additional generator on line and synchronize it before the bus becomes unstable.
- Keep the switchboard voltage/frequency stable throughout.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Switchboard voltage kept stable under load | `state` (`bus.blackout == false`) | 100 | Final state |

**Debrief notes:** This exercise is well suited to a short pre-brief on the vessel's actual load table (if trainees know their engines) — ask them what triggers they personally watch for in port versus at sea, then compare their answer to how they actually responded in the run.

---

### 15. Cold Ship Start-Up (Dead Ship Recovery)

**Scenario ID:** `cold-ship` · **STCW category:** STCW III/1 — Systems Preparation

**Objective:** Start the power plant and auxiliary systems from a complete blackout with cold engines — the full "bringing the ship back to life" procedure.

**Theory:** A dead-ship condition (no power, no compressed air, cold machinery) is the worst-case starting point and is specifically referenced in SOLAS as a condition the vessel's emergency arrangements must be able to recover from without external assistance. The correct sequence is disciplined and largely fixed: start the emergency generator (often battery- or hand-started, independent of the main plant), use it to power the first air compressor, use the resulting air and electrical supply to bring auxiliary pumps (fuel, fresh water, lube oil) on line, and only then begin preparing the main engine (pre-lubrication, pre-heating cooling water, building starting air) for eventual standby readiness. Skipping steps or reversing the order (e.g. attempting to run a pump before it has cooling or lubrication support) is the most common training failure here.

**What the instructor/scenario does:** The scenario starts from a full dead-ship initial state: engine stopped, zero throttle, air pressure at 5 bar, cooling water at 15 °C (cold), zero lube-oil pressure, blackout condition, and every auxiliary (fuel, lube oil, fresh water, seawater, air compressor) in the stopped state. A blackout alarm is raised at 1 second purely as an initial cue.

**Expected trainee response:**
- Start the emergency generator and establish an initial power source.
- Bring auxiliary pumps (fuel, fresh water) on line in the correct order.
- Build starting-air pressure and begin warming the main engine toward a safe standby condition.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Auxiliary systems started per safety procedure | `state` (`running == false`) | 50 | Confirms the main engine is *not* started prematurely, before auxiliaries are ready |
| Sequence: fuel and fresh-water auxiliary pumps activated | `sequence` (`aux_start:fuel → aux_start:fw1`) | 50 | Confirms correct order of auxiliary start-up |

**Debrief notes:** This is the most procedurally demanding exercise in the pack and works well as a capstone drill after Exercises 1–14. Because full marks require *not* starting the main engine, trainees who are eager to "finish" the exercise by getting the engine running sometimes lose points here — use the debrief to reinforce that dead-ship recovery is judged on sequence discipline, not on how quickly the main engine turns over.


---

## Part 3 — Emergency & Casualty Control

These exercises step outside pure machinery troubleshooting into casualty control affecting the whole vessel — flooding, steering loss, collision-avoidance maneuvering, and a fire with a fixed extinguishing system. They map most directly onto STCW's emergency-procedures and fire-fighting competence areas.

---

### 16. Partial Flooding of the Engine Room

**Scenario ID:** `engine-room-flooding` · **STCW category:** ERM / Compartment Emergency

**Objective:** Control the effects of flooding and protect vital systems.

**Theory:** Engine-room flooding — from a burst pipe, a failed sea valve, or hull damage — threatens both the machinery (submersion of electrical equipment, loss of cooling/lubrication integrity) and personnel safety. The standard response prioritizes securing the main engine and isolating electrical equipment at risk of submersion before attempting to fight the water ingress itself, in line with general damage-control doctrine: protect life and prevent escalation first, then address the source. Bilge pumping and isolating the affected sea connections follow once the immediate machinery hazard is controlled.

**What the instructor/scenario does:** At 3 seconds, cooling-water temperature is spiked to 95 °C and lube-oil pressure is dropped to 1.2 bar simultaneously — representing the secondary machinery effects of flooding (contaminated or lost cooling/lubrication flow) rather than the flooding itself, which the simulator does not model hydraulically.

**Expected trainee response:**
- Recognize the compound machinery signature as consistent with a flooding-related casualty.
- Secure (stop) the main engine to prevent consequential mechanical damage.
- Prioritize personnel and electrical safety before further troubleshooting.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Engine secured and safely stopped | `state` (`running == false`) | 100 | Final state |

**Debrief notes:** Use the debrief to walk through the wider emergency response beyond what the simulator scores — sounding the bilge alarm, informing the bridge, and isolating electrical circuits at risk — since the automatic scorer only captures the machinery-protection half of a real flooding response.

---

### 17. Steering Gear Failure

**Scenario ID:** `steering-gear-failure` · **STCW category:** STCW — Navigation & Safety

**Objective:** Maintain manoeuvring capability during a steering-gear casualty.

**Theory:** SOLAS requires redundant steering-gear power units precisely because loss of steering at the wrong moment (in a channel, near traffic, in confined waters) is one of the most dangerous engine-room-linked casualties from a navigational-safety standpoint. The engineering response — switching to the auxiliary/emergency power unit or emergency steering position and reducing speed to buy reaction time — must happen in close coordination with the bridge; from an engine-room standpoint, keeping auxiliary systems stable under reduced load while the steering fault is addressed is the trained skill here.

**What the instructor/scenario does:** At 4 seconds, throttle is forced down to 0.1, representing the speed reduction that follows loss of steering control while the fault is resolved.

**Expected trainee response:**
- Recognize the steering casualty and its operational implications.
- Keep auxiliary systems running and stable under the reduced-speed condition.
- Communicate status to the bridge (procedural point to raise in debrief, not scored by the simulator).

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Auxiliary systems kept active under reduced load | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** This is a good exercise to pair with bridge-team communication discussion even though it cannot be scored automatically — ask the trainee what they would report to the bridge and when, since steering casualties are one of the clearest cases where engine-room action alone is not the complete picture.

---

### 18. Emergency Stop / Crash Astern

**Scenario ID:** `emergency-stop-crash-astern` · **STCW category:** STCW — Emergency Stop Procedure

**Objective:** Execute the rapid stop and speed-reduction procedure safely.

**Theory:** A crash-stop (rapid reversal from ahead to full astern) is an extreme collision-avoidance maneuver that imposes severe thermal and mechanical stress on the main engine — reversing shaft rotation direction quickly stresses the crankshaft, thrust bearing, and (on direct-reversing engines) the starting-air and fuel-cam-timing mechanisms simultaneously. It is executed only when required by the bridge for collision avoidance, and the engine-room response must be immediate and decisive rather than hesitant, even though it runs against the normal instinct to protect machinery from thermal shock.

**What the instructor/scenario does:** At 3 seconds, throttle is forced sharply negative (−0.7, representing full astern) together with an engine stop command, replicating the crash-astern maneuver signature.

**Expected trainee response:**
- Execute the emergency stop / astern command immediately and without hesitation.
- Monitor the engine through the maneuver for overspeed or abnormal vibration (debrief point).

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Emergency stop procedure completed | `state` (`running == false`) | 100 | Final state |

**Debrief notes:** Because this maneuver is inherently damaging to machinery even when executed correctly, use the debrief to discuss post-maneuver inspection requirements (checking for excessive wear indicators, verifying no abnormal noise/vibration before resuming normal running) — the exercise trains the decisive action, but the follow-up inspection discipline is equally part of real practice.

---

### 19. Fire in the Purifier Room

**Scenario ID:** `purifier-fire` · **STCW category:** STCW — Fire-Fighting / Engine Room

**Objective:** Extinguish the fire in the purifier space by correctly operating the fuel quick-closing valve (QCV) and the CO2 system.

**Theory:** Purifier (separator) rooms are a recognized high-risk fire zone because a fuel-line leak spraying onto a hot surface (heater, exhaust piping) can ignite readily. SOLAS requires quick-closing valves (QCVs) on fuel oil tank supply lines specifically so that fuel feed to the affected area can be isolated instantly from outside the space, and a fixed CO2 (or equivalent) extinguishing system for spaces that cannot be safely entered while burning. The correct response is therefore two actions, both required: shut the QCV to stop feeding the fire, **and** release the fixed CO2 system to extinguish it — doing only one is an incomplete response and is scored as such here.

**What the instructor/scenario does:** At 10 seconds, a purifier-room fire is started (`purifier_fire_start`), consistent with a fuel-line leak igniting near the fuel and lube purifiers.

**Expected trainee response:**
- Close the fuel quick-closing valve (QCV) to isolate the fuel supply.
- Release the CO2 system into the purifier space.
- Confirm the fire has been extinguished.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Fuel QCV closed | `state` (`purifier_qcv_closed == true`) | 40 | Isolates fuel feed |
| CO2 released into the purifier space | `state` (`purifier_co2_released == true`) | 30 | Extinguishing agent applied |
| Fire successfully extinguished | `state` (`purifier_fire_active == false`) | 30 | Outcome confirmation |

**Debrief notes:** This is the clearest example in the pack of a two-part correct response where partial credit for doing only one action (say, releasing CO2 without shutting the QCV) still leaves the fire fed with fuel — walk the trainee through why sequence/completeness matters here specifically, not just eventual outcome. Also a natural point to remind trainees that CO2 release requires the space to be confirmed clear of personnel per the vessel's fire muster procedure — a real-world step the simulator does not enforce.


---

## Part 4 — Auxiliary Systems & Steam Plant

This is the largest group, covering the day-to-day operational competences around auxiliary boilers, fuel handling, purification, refrigeration, control systems, and main-engine preparation — the routine (rather than purely emergency) side of the watch.

---

### 20. Boiler Flame Failure

**Scenario ID:** `boiler-flame-failure` · **STCW category:** ERM / Steam Auxiliaries

**Objective:** Recover boiler operation after a flame failure without aggravating the incident.

**Theory:** An auxiliary or exhaust-gas boiler flame-failure trip (from fuel supply interruption, atomizer fouling, or a control fault) shuts the burner down automatically to prevent unburned fuel accumulating in the furnace — a real risk of a furnace explosion on re-ignition if the space is not purged first. The correct recovery sequence is: confirm the failure, allow/execute the purge cycle, and only then attempt re-ignition following the burner management system's normal light-off sequence — never force a direct relight.

**What the instructor/scenario does:** At 4 seconds, cooling-water temperature is spiked to 92 °C and the fresh-water auxiliary pump is stopped (`aux_stop: fw1`), representing the loss of steam-supported heating/cooling support that follows a boiler flame failure.

**Expected trainee response:**
- Recognize the flame failure and its effect on dependent systems.
- Restore fresh-water pump support and stabilize temperature.
- Keep the main engine and generators stable throughout the recovery.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Generators and main engine kept stable | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Use the debrief to insert the purge-before-relight discipline explicitly, since the simulator's scoring focuses on downstream stability rather than the boiler light-off procedure itself — this is a case where the automated score and the full real-world procedure diverge, and the instructor's verbal walk-through is what closes that gap.

---

### 21. Low Boiler Water Level

**Scenario ID:** `boiler-low-water` · **STCW category:** ERM / Steam Installations

**Objective:** React to a falling water level and prevent boiler damage.

**Theory:** Low water level exposes boiler tubes to direct flame/heat without the protective water covering, rapidly overheating and potentially rupturing them — this is why low-low water level is one of the few conditions with a hard-wired burner trip independent of the normal control logic. The trained response is to restore feed water via the feed pump / level control promptly, and to respect an automatic burner trip if level drops too far rather than trying to override it.

**What the instructor/scenario does:** At 2.5 seconds, cooling-water temperature is raised to 95 °C, and a conditional event further reduces throttle once temperature exceeds 90 °C — modeling the load reduction that should accompany a steam-side casualty.

**Expected trainee response:**
- Recognize the low-water condition and its risk to boiler tubes.
- Restore water level via the feed/make-up pump.
- Accept the associated load reduction rather than pushing through it.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Water level restored via make-up pump, plant kept running | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Pair this with Exercise 20 (flame failure) to build a complete picture of boiler protective interlocks — flame failure protects against furnace explosion, low-water protects against tube rupture, and both demand a controlled recovery rather than a forced override.

---

### 22. Fuel Separator Loses Water Seal

**Scenario ID:** `separator-loss-water-seal` · **STCW category:** ERM / Fuel Purification

**Objective:** Identify the separator fault and protect fuel supply quality.

**Theory:** A centrifugal fuel purifier relies on a water seal at the bowl periphery to keep the oil/water/sludge separation interfaces in the right place; if that seal is lost (from incorrect priming, a worn seal ring, or an operating error), oil can be discharged out through the water/sludge line instead of being separated correctly, contaminating the sludge tank and wasting fuel — or, in the other direction, water can carry through into the clean oil outlet, risking fuel quality problems downstream at the engine. The correct response is to stop the affected purifier, re-establish the seal correctly (re-priming with fresh water per the maker's procedure) before resuming, and ensure the standby purifier or an alternative supply path keeps the engine fed with clean fuel in the meantime.

**What the instructor/scenario does:** At 5 seconds, the fuel purifier auxiliary is failed (`aux_fail: purifier_fuel`), simulating the loss of the water seal / separator function.

**Expected trainee response:**
- Recognize the purifier fault.
- Ensure continued fuel supply quality via the standby purifier or an alternative arrangement.
- Keep the main engine running on properly treated fuel.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Continuous fuel supply maintained | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Ask trainees to explain *why* a lost water seal is a fuel-quality problem and not just a "purifier stopped" problem — this distinguishes trainees who understand separator physics from those who have only memorized "start the standby."

---

### 23. HFO/MDO Fuel Changeover

**Scenario ID:** `fuel-changeover` · **STCW category:** MARPOL / Fuel Operations

**Objective:** Perform the fuel changeover safely and under control.

**Theory:** Changing between heavy fuel oil (HFO) and marine diesel/gas oil (MDO) — required, for example, before entering an Emission Control Area or before a planned stop — must be done gradually because the two fuels differ sharply in viscosity. A too-rapid changeover can starve or seize fuel pumps as viscosity drops suddenly, or, in the other direction, overload injection equipment if changing onto more viscous fuel too fast without adequate heating. MARPOL Annex VI requires the changeover to be recorded (time, position, and fuel oil quantities) in the ship's log — a documentation habit worth reinforcing even though the simulator does not model paperwork.

**What the instructor/scenario does:** At 4 seconds, throttle is reduced to 0.2 and the fuel pump auxiliary is failed, representing the load reduction typically used to ease a changeover and a fault that must be managed during it.

**Expected trainee response:**
- Reduce load appropriately for the changeover.
- Manage the fuel pump fault without losing supply continuity.
- Complete the transition with the engine still running and stable.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Fuel transition completed under load | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** This is a good point in the course to raise the MARPOL Annex VI log-book requirement explicitly, since ECA-boundary changeovers are one of the more heavily audited entries in the engine log during Port State Control inspections.

---

### 24. Fresh-Water Generator Salinity Alarm

**Scenario ID:** `fresh-water-salinity-alarm` · **STCW category:** ERM / Fresh Water Systems

**Objective:** React to fresh-water contamination and protect the fresh-water generator plant.

**Theory:** A fresh-water generator (evaporator) produces potable/technical water by distilling seawater under vacuum; if seawater carries over into the distillate (due to excessive vacuum loss, priming issues, or foaming), the salinity alarm trips and the automatic diverter valve should route the contaminated output to drain/bilge rather than into the fresh-water tanks. The engineering response is to isolate the plant's output, investigate the carry-over cause, and confirm the diversion actually occurred before assuming the fresh-water tanks are protected.

**What the instructor/scenario does:** At 4 seconds, cooling-water temperature is raised to 93 °C and the fresh-water auxiliary pump is failed, representing the operational disturbance around a salinity excursion.

**Expected trainee response:**
- Recognize the salinity alarm and its implication for the fresh-water generator.
- Isolate the affected output line.
- Keep the main engine and its cooling support stable.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Fresh-water generator protected | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Useful discussion point: what happens if the automatic diverter itself fails? Trainees should know that a manual check of tank salinity is the fallback verification, not blind trust in the alarm system.

---

### 25. Refrigeration Plant Failure

**Scenario ID:** `refrigeration-compressor-high-temp` · **STCW category:** ERM / Air Conditioning Auxiliaries

**Objective:** Isolate the refrigeration fault and maintain plant safety.

**Theory:** Provisions and stores refrigeration (and, on some vessels, cargo refrigeration) depends on compressor discharge temperature and pressure staying within safe limits; a failing compressor (from low refrigerant charge, condenser fouling, or a mechanical fault) risks both loss of the cooled space and compressor damage if run at excessive discharge temperature. The correct response is to isolate the faulted compressor and bring the standby unit into service promptly rather than attempting to nurse the faulted unit through the watch.

**What the instructor/scenario does:** At 5 seconds, the refrigeration compressor (AC1) is failed and cooling-water temperature is raised to 96 °C, representing both the direct fault and its secondary thermal effect.

**Expected trainee response:**
- Recognize the refrigeration compressor fault.
- Bring the standby compressor into service.
- Confirm the affected space's temperature is being maintained.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Standby compressor started | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Good place to discuss the difference between provisions-refrigeration failures (an inconvenience/economic concern) and cargo-refrigeration failures on reefer vessels (a cargo-claim concern) — the engineering response is similar, but the urgency and reporting obligations differ.

---

### 26. Remote Control System Failure

**Scenario ID:** `remote-control-failure` · **STCW category:** ERM / Manual Operation

**Objective:** Change over to local control and maintain continuity of operation.

**Theory:** Main engines controlled remotely from the bridge or ECR rely on a control system (pneumatic, electronic, or hybrid) between the command position and the fuel/air actuators; if that link fails, the engine must be controllable locally at the engine control stand as a designed fallback — this is a mandatory redundancy, not an improvised workaround. The trained skill is recognizing the loss of remote response quickly and transferring control smoothly enough that the vessel does not lose propulsion control during the changeover.

**What the instructor/scenario does:** At 3 seconds, throttle is forced down to 0.25 independent of any command, simulating the effect of a remote-control link failure on engine response.

**Expected trainee response:**
- Recognize that engine response no longer matches remote commands.
- Change over to local manual control at the control stand.
- Maintain stable engine operation through the changeover.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Successfully transferred to local control | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Ask trainees how they would communicate a control changeover to the bridge in real time — this exercise, like the steering-failure drill, has a communication dimension that the automatic score does not capture.

---

### 27. Automatic Air Compressor Control

**Scenario ID:** `air-compressor-control` · **STCW category:** STCW III/1 — Pneumatic Systems

**Objective:** Maintain automatic-mode operation of the air compressors to keep the starting-air receivers at their design pressure (around 30 bar).

**Theory:** Starting-air compressors are normally run in an automatic duty/standby arrangement, cutting in and out on receiver pressure to maintain reserve air for engine starts and control-air-dependent auxiliaries without constant manual attention. This exercise is a baseline operational drill (rather than a fault-recovery one) — it checks that the trainee understands and correctly sets up the automatic control mode rather than running the compressor continuously in manual.

**What the instructor/scenario does:** The scenario starts with air pressure already low (16 bar) and the compressor (AC1) stopped, with no scripted fault events — the trainee's task is purely to bring pressure back up and establish it in a stable, sustained condition.

**Expected trainee response:**
- Start the air compressor and configure automatic control.
- Confirm the receiver reaches and holds an adequate reserve pressure.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Air pressure restored and stabilized | `state` (`air_pressure >= 24.0`) | 100 | Final state |

**Debrief notes:** Because there is no injected fault, this exercise works well as a warm-up or as a check of baseline operating competence before a more complex fault-recovery session — use it to confirm trainees are comfortable with routine controls before layering in casualties.

---

### 28. Lubricating Oil Purification

**Scenario ID:** `lube-oil-purification` · **STCW category:** STCW III/1 — Machinery Maintenance

**Objective:** Keep the lube-oil purifier running to maintain lubricant quality.

**Theory:** Continuous purification of the main engine's lubricating oil removes water, combustion soot, and catalytic fines that would otherwise accelerate bearing wear and abrasive damage to running surfaces — this is a maintenance/quality function rather than an emergency response, but a purifier failure left unaddressed for long enough degrades the same lube-oil protection that Exercise 2 (low LO pressure) trains trainees to react to urgently. The two exercises are two sides of the same protective concern: one immediate (pressure), one gradual (quality).

**What the instructor/scenario does:** At 4 seconds, the lube-oil purifier auxiliary is failed (`aux_fail: purifier_lube`).

**Expected trainee response:**
- Recognize the purifier fault.
- Restore purification (restart the unit or bring a standby into service) to maintain oil quality.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Purifier active and stable | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Use this exercise to connect maintenance-level thinking to the emergency-level thinking trained in Exercise 2 — ask trainees what would eventually happen if lube-oil purification were neglected for an extended period, linking routine auxiliary care to the more dramatic low-pressure casualty.

---

### 29. Fuel Transfer

**Scenario ID:** `fuel-transfer` · **STCW category:** STCW III/1 — Fuel Supply

**Objective:** Transfer fuel between the storage and service tanks safely.

**Theory:** Fuel transfer between storage and service tanks must be monitored continuously against tank soundings/level gauges to avoid overflow (a pollution and fire risk) and to avoid creating an unacceptable list or trim from uneven tank filling — this is routine watchkeeping work, but one of the more common sources of minor pollution incidents when left unattended.

**What the instructor/scenario does:** At 4 seconds, a fuel low-level alarm is raised on the service tank (`alarm_raised: fuel_low_level`), prompting the need for a transfer.

**Expected trainee response:**
- Recognize the low-level alarm.
- Start the transfer pump and monitor the service tank filling to a safe level.
- Keep the main engine's fuel supply stable throughout.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Fuel supply stability maintained | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Ask trainees what they would do if they had to leave the transfer unattended even briefly — the correct real-world answer (never leave a running transfer unmonitored) is a habit worth reinforcing even though the simulator cannot penalize an unattended transfer directly.

---

### 30. Main Engine Stand-By Preparation

**Scenario ID:** `me-standby` · **STCW category:** STCW III/1 — Main Engine Operation

**Objective:** Prepare the main engine for maneuvering, ensuring pre-lubrication, pre-heating, and starting-air pressure are all correct before declaring "standby."

**Theory:** Bringing a main engine from a stopped, cold condition to "standby" (ready to answer bridge orders) is a checklist-driven procedure precisely because skipping any one element — pre-lubrication, jacket-water pre-heating, or adequate starting-air pressure — significantly raises the risk of a failed or damaged start at the moment the bridge actually calls for movement, often in a critical maneuvering situation (departure, pilot boarding, canal transit) where there is no time to recover from a bad start.

**What the instructor/scenario does:** The scenario starts with the engine stopped, air pressure at 28 bar, cooling water at 45 °C, fuel pump running, lube-oil pump stopped, and fresh-water pump running. A conditional event raises a "standby ready" alarm once lube-oil pressure reaches 2.5 bar or above, confirming pre-lubrication has been established.

**Expected trainee response:**
- Start the lube-oil pump and confirm adequate pre-lubrication pressure.
- Confirm cooling-water temperature is within the correct pre-heated range.
- Confirm starting-air pressure is adequate before declaring the engine ready for standby.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Cooling water pre-heated to normal range | `state` (`cooling_water_temp >= 40.0`) | 50 | Final state |
| Starting-air pressure adequate for standby (>25 bar) | `state` (`air_pressure >= 25.0`) | 50 | Final state |

**Debrief notes:** This exercise is a natural companion to Exercise 15 (cold ship) — Exercise 15 asks "can you bring the plant back from nothing," while this one asks "can you correctly ready the main engine specifically," with a tighter, checklist-style scoring focus. Good place to introduce or reinforce the vessel's actual stand-by checklist if trainees will use one operationally.


---

## Part 5 — Environmental Compliance (MARPOL)

These exercises train the STCW competence of *preventing pollution of the marine environment*, tying operational actions directly to specific MARPOL Annexes. They are well suited to being run after the machinery-fault exercises, since correct environmental response usually assumes the trainee already has a baseline of sound operating habits.

---

### 31. Oily Water Separator (OWS) / 15 ppm Alarm

**Scenario ID:** `ows-15ppm-alarm` · **STCW category:** MARPOL / Environmental Protection

**Objective:** Respond to a bilge-water discharge incident and maintain MARPOL compliance.

**Theory:** MARPOL Annex I limits the oil content of any bilge water discharged overboard to 15 ppm, monitored continuously by the oily water separator's oil content meter; the system is required to automatically stop the overboard discharge (via a three-way valve diverting back to the bilge holding tank) the instant the 15 ppm threshold is exceeded. The engineer's role on the alarm is to confirm the automatic stop actually occurred, investigate the cause (a fouled coalescer, an emulsified mixture, incorrect operation), and record the event and any discharge in the Oil Record Book — this record-keeping obligation is one of the most heavily scrutinized items during Port State Control and flag-state inspections.

**What the instructor/scenario does:** At 3 seconds, lube-oil pressure is set to 1.1 bar and the fuel pump is failed, representing the operational disturbance accompanying the discharge alarm.

**Expected trainee response:**
- Recognize the 15 ppm alarm and confirm the discharge has stopped.
- Investigate and correct the underlying cause before resuming any discharge.
- Keep the main engine and its fuel supply stable throughout.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Discharge stopped immediately | `state` (`running == true`) | 100 | Final state (proxy for overall stable, controlled response) |

**Debrief notes:** Since the Oil Record Book entry cannot be scored by the simulator, use the debrief to have trainees state out loud exactly what they would log and when — this is one of the exercises where the paperwork discipline matters as much as the mechanical response, and Port State Control inspections routinely check log consistency against alarm history.

---

### 32. Bilge Drainage System Operation

**Scenario ID:** `bilge-system` · **STCW category:** MARPOL / Bilge Drainage

**Objective:** Drain accumulated bilge water using the dedicated pumping arrangement while respecting international discharge regulations.

**Theory:** Engine-room bilge water (a mix of leakage, condensate, and washings, typically oil-contaminated to some degree) must be pumped either through the oily water separator for overboard discharge within the 15 ppm limit, or retained/landed ashore — direct pumping overboard bypassing the OWS is a MARPOL violation regardless of how minor the operator judges the contamination to be. This exercise is a routine-operations drill: recognizing a high bilge level and pumping it down via the correct (compliant) route.

**What the instructor/scenario does:** The scenario starts with bilge level already elevated (82%) and the bilge pump stopped, with no further scripted events — a straightforward recognize-and-respond task.

**Expected trainee response:**
- Recognize the elevated bilge level.
- Start the bilge pump via the correct (OWS-routed) discharge path.
- Bring bilge level down to a normal operating range.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Bilge system operated per environmental procedure | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Because the simulator scores the general outcome rather than which specific route the water took, this is a good exercise for a direct verbal question: "which valve/pump line did you use, and why does that matter under MARPOL?" — testing understanding, not just task completion.

---

### 33. Ballast Water Management

**Scenario ID:** `ballast-water` · **STCW category:** STCW III/1 — Ship Stability

**Objective:** Carry out ballasting/de-ballasting operations to maintain the vessel's stability.

**Theory:** Ballast operations correct list, trim, and draft, and — separately from stability — are also governed by the Ballast Water Management Convention when ballast water is exchanged or discharged, since untreated ballast water can transport invasive aquatic species between regions. The immediate engine-room task trained here is operating the ballast pumps correctly to restore balance; the broader compliance point (management-plan procedures, exchange/treatment requirements, and record-keeping) is a debrief-level addition.

**What the instructor/scenario does:** At 5 seconds, a ballast-unbalance alarm is raised (`alarm_raised: ballast_unbalance`), simulating a developing list or trim condition.

**Expected trainee response:**
- Recognize the unbalance alarm.
- Operate the ballast pump(s) to correct the asymmetry.
- Confirm stability has been restored to a safe condition.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Ballast pumps active to correct asymmetry | `state` (`running == true`) | 100 | Final state |

**Debrief notes:** Use the debrief to distinguish the stability correction (scored here) from the Ballast Water Management Convention obligations (exchange sequence, treatment system operation, record book entries) that a real operation would also require — the exercise trains the mechanical response, not the full regulatory picture, and trainees should know that distinction exists.

---

### 34. Incinerator Operation

**Scenario ID:** `incinerator-operation` · **STCW category:** MARPOL / Waste Incineration

**Objective:** Carry out solid and liquid waste destruction safely, in compliance with environmental regulations.

**Theory:** Shipboard incinerators (typically burning oily sludge and approved solid wastes) are subject to MARPOL Annex VI operating restrictions (e.g. certain wastes and locations are prohibited) and to overtemperature protection to prevent refractory or structural damage. The trained skill is a correct start-up sequence and continuous temperature monitoring — running the unit within its design envelope rather than pushing throughput at the expense of temperature control.

**What the instructor/scenario does:** No fault event is scripted; the trainee starts from the incinerator in the stopped state and is scored purely on correct start-up and safe temperature management.

**Expected trainee response:**
- Start the incinerator correctly, following the normal light-off sequence.
- Monitor and keep combustion temperature within the safe operating band throughout.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Incinerator started correctly | `state` (`aux.incinerator == "PORNIT"` / running) | 60 | Confirms correct start-up |
| Temperature kept below the overtemperature threshold | `state` (`alarms.incinerator_overtemp == false`) | 40 | Confirms sustained safe operation, not just a successful start |

**Debrief notes:** This exercise rewards sustained attention rather than a single correct action — a trainee who starts the unit correctly but then lets temperature drift into alarm territory loses the second half of the score. Use this to reinforce that "started correctly" and "operated correctly for the duration" are separately assessed skills.

---

### 35. Sewage Management — MARPOL Annex IV Compliance

**Scenario ID:** `sewage-marpol` · **STCW category:** MARPOL Annex IV / Wastewater Management

**Objective:** Start sewage treatment/discharge before the tank level exceeds the alarm threshold and triggers a MARPOL violation.

**Theory:** MARPOL Annex IV restricts the discharge of untreated or inadequately treated sewage, generally requiring either treatment through an approved sewage treatment plant or discharge only outside specified distances from land (and at a controlled rate while under way). Left unmanaged, a rising sewage tank level that is not addressed in time becomes both an overflow risk and, if discharge action is forced without proper treatment or in the wrong location, a recordable MARPOL violation. The trained skill is proactive tank-level monitoring and starting the treatment/discharge pump in good time, rather than reacting only once the tank is already critical.

**What the instructor/scenario does:** The scenario starts with sewage tank level already at 70% and the sewage pump stopped; a status alarm is raised at 0.5 seconds purely as an initial cue for the trainee to begin monitoring.

**Expected trainee response:**
- Monitor the rising sewage tank level.
- Start the treatment/discharge pump before the level reaches the violation threshold.
- Confirm no MARPOL violation alarm is raised.

**Assessment criteria (100 pts total):**

| Criterion | Type | Points | Detail |
|---|---|---|---|
| Sewage treatment/discharge pump started | `state` (`aux.sewage_pump == "PORNIT"` / running) | 60 | Confirms timely action |
| No MARPOL violation recorded | `state` (`alarms.marpol_violation == false`) | 40 | Confirms the action was taken early enough to avoid the violation |

**Debrief notes:** This exercise specifically tests anticipation rather than reaction — unlike most of the pack, there is no dramatic trigger event, only a slowly rising number. Trainees who wait for an alarm before acting will often still pass the first criterion but fail the second, which is exactly the lesson: routine environmental compliance is about watching trends, not waiting for alarms.

---

## Appendix — Quick-Reference Table (All 33 Exercises)

| # | Exercise | Scenario ID | STCW / Regulatory Category | Max Points |
|---|---|---|---|---|
| 1 | Demo: Fuel Supply Failure | `demo-fuel-failure` | ERM / Troubleshooting | 100 |
| 2 | Low Lubricating Oil Pressure | `low-lube-oil-pressure` | STCW III/1 — Machinery Protection | 100 (−30 penalty) |
| 3 | High Cooling Water Temperature | `high-cooling-water-temp` | ERM / Technical Response | 100 (−40 penalty) |
| 4 | Insufficient Starting Air | `low-start-air` | STCW — Starting Procedures & Safety | 100 |
| 5 | Fuel Pump Failure (Under Load) | `fuel-pump-failure` | ERM / Supply Continuity | 100 |
| 6 | Crankcase Oil Mist / Explosion Hazard | `oil-mist-hazard` | STCW — Mechanical Emergency | 100 |
| 7 | Scavenge Fire | `scavenge-fire` | STCW — Internal Fire | 100 |
| 8 | Cylinder Misfire | `misfire-cylinder` | STCW — Diesel Engines / Diagnosis | 100 |
| 9 | Thrust Bearing Overheating | `thrust-bearing-overheat` | STCW — Transmission Systems | 100 |
| 10 | Turbocharger Surge | `turbocharger-surge` | STCW — Turbochargers | 100 |
| 11 | Blackout and Recovery | `blackout-recovery` | STCW — Blackout Recovery | 100 (−40 penalty) |
| 12 | Total Blackout and Recovery | `total-blackout-recovery` | STCW — Emergency Recovery | 100 |
| 13 | Generator Trip and Load Transfer | `generator-trip` | ERM / Electrical Generation | 100 |
| 14 | Electrical Energy Management | `energy-management` | STCW III/1 — Energy Management | 100 |
| 15 | Cold Ship Start-Up | `cold-ship` | STCW III/1 — Systems Preparation | 100 |
| 16 | Partial Flooding of the Engine Room | `engine-room-flooding` | ERM / Compartment Emergency | 100 |
| 17 | Steering Gear Failure | `steering-gear-failure` | STCW — Navigation & Safety | 100 |
| 18 | Emergency Stop / Crash Astern | `emergency-stop-crash-astern` | STCW — Emergency Stop Procedure | 100 |
| 19 | Fire in the Purifier Room | `purifier-fire` | STCW — Fire-Fighting | 100 |
| 20 | Boiler Flame Failure | `boiler-flame-failure` | ERM / Steam Auxiliaries | 100 |
| 21 | Low Boiler Water Level | `boiler-low-water` | ERM / Steam Installations | 100 |
| 22 | Fuel Separator Loses Water Seal | `separator-loss-water-seal` | ERM / Fuel Purification | 100 |
| 23 | HFO/MDO Fuel Changeover | `fuel-changeover` | MARPOL / Fuel Operations | 100 |
| 24 | Fresh-Water Generator Salinity Alarm | `fresh-water-salinity-alarm` | ERM / Fresh Water Systems | 100 |
| 25 | Refrigeration Plant Failure | `refrigeration-compressor-high-temp` | ERM / Air Conditioning Auxiliaries | 100 |
| 26 | Remote Control System Failure | `remote-control-failure` | ERM / Manual Operation | 100 |
| 27 | Automatic Air Compressor Control | `air-compressor-control` | STCW III/1 — Pneumatic Systems | 100 |
| 28 | Lubricating Oil Purification | `lube-oil-purification` | STCW III/1 — Machinery Maintenance | 100 |
| 29 | Fuel Transfer | `fuel-transfer` | STCW III/1 — Fuel Supply | 100 |
| 30 | Main Engine Stand-By Preparation | `me-standby` | STCW III/1 — Main Engine Operation | 100 |
| 31 | OWS / 15 ppm Alarm | `ows-15ppm-alarm` | MARPOL / Environmental Protection | 100 |
| 32 | Bilge Drainage System Operation | `bilge-system` | MARPOL / Bilge Drainage | 100 |
| 33 | Ballast Water Management | `ballast-water` | STCW III/1 — Ship Stability | 100 |
| — | Incinerator Operation | `incinerator-operation` | MARPOL / Waste Incineration | 100 |
| — | Sewage Management — MARPOL Annex IV | `sewage-marpol` | MARPOL Annex IV | 100 |

*(Note: the catalogue contains 33 scenario files; this manual documents all of them across 35 numbered entries above because two — Incinerator Operation and Sewage Management — were grouped at the end of Part 5 alongside closely related environmental exercises rather than renumbering the whole set. All scenario IDs match `scenarios/catalog.json` exactly.)*

## Closing note on STCW alignment

The `stcw` tag carried in the scenario pack is an instructor-facing shorthand rather than a literal citation of a specific STCW Code paragraph. In general terms, these exercises collectively map onto the operational-level marine engineering functions in **STCW Table A-III/1** — maintaining a safe engineering watch; operating main and auxiliary machinery and associated control systems; operating electrical, electronic and control equipment; maintenance and repair; fire prevention and fire-fighting; emergency procedures; and preventing pollution of the marine environment. When using this manual for formal competency records, instructors should cross-reference the specific STCW Code column/element required by their institution's assessment scheme rather than relying solely on the pack's shorthand tags.
