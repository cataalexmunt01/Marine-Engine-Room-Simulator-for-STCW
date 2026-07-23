# Physical and Mathematical Theory of the Model — STCW Engine Room Simulator

Technical reference document. All formulas below are transcribed directly from engine\_core.py, scenario\_engine.py, exercise\_eval.py, and coach\_engine.py — they are the exact description of the running code, not retrospective approximations.

## 1\. Model philosophy

The physics core (engine\_core.py) is adapted and simplified from the NTNU academic project ship\_in\_transit\_simulator (Børge Rokseth), from which the basic classes PidController and EulerInt were retained, along with the shaft equilibrium equation (shaft\_eq). All ship hydrodynamics (surge/sway/yaw, environment, course) have been removed — for an ERS, only the propulsion and power generation system is of interest.

The default main engine is a Wärtsilä 6L26 (2160 kW); the diesel generators are Baudouin 6M26.3 (510 kW each) — actual specific consumption coefficients from the NTNU source.

All simulation uses explicit Euler integration with fixed step dt (default 0.1 s, 10 Hz):

x(t \+ dt) \= x(t) \+ dx/dt · dt

EulerInt class does exactly this: integrate(x, dx) \= x \+ dx·dt.

## 2\. Dynamics of the main engine and propeller shaft

The model explicitly separates the engine's own speed (crankshaft, ω\_motor) from the propeller shaft speed (ω\_ax), coupled via a slip clutch. This allows the engine to accelerate/decelerate quickly at idle (low inertia) even when disengaged, while the propeller shaft has high inertia and responds slowly.

### 2.1 Speed regulator (governor)

Classic PID controller:

error \= setpoint − measurement

error\_i \+= error · dt  (bounded to \[−1, 1\])

d\_error \= (error − previous\_error) / dt

output \= Kp·error \+ Kd·d\_error \+ Ki·error\_i

For the main engine: Kp \= 0.15, Ki \= 0.02, Kd \= 0\. The target is target\_omega \= effective\_throttle · ω\_nominal, the measurement is ω\_engine. The output (load\_cmd, saturated to \[−1, 1\]) becomes the requested load percentage.

### 2.2 Main engine torque

torque\_me \= min( |load\_perc| · P\_available / (ω\_motor \+ 0.1),

                  P\_available / 5 · π/30 ) · sign(load\_perc)

The term ω\_motor \+ 0.1 avoids division by zero at zero speed; the second term limits maximum torque at start-up (approximation of a real diesel torque curve).

### 2.3 Engine shaft equation (crankshaft)

eq\_me \= (torque\_me − friction\_me · ω\_motor) / gear\_ratio\_me

d(ω\_motor)/dt \= (eq\_me − clutch\_couple) / J\_motor

J\_motor (engine\_inertia, default 2500\) — the engine's own inertia, much smaller than the propeller's inertia, which is why the disengaged engine revs quickly.

### 2.4 Clutch (pneumatic coupling)

The clutch position does not instantly follow the command — it approaches progressively:

position(t+dt) \= position(t) ± dt / ramp\_time

with ramp\_time \= 4.0 s when engaging (ENGAGE\_RAMP\_S) and 1.5 s when disengaging (DISENGAGE\_RAMP\_S).

Torque transmitted through the clutch:

slip \= ω\_motor − ω\_axle

clutch\_torque \= position · torque\_capacity · sat( slip · 0.08, −1, 1 )

Engagement interlock: the engagement command is rejected if the engine is not running OR its speed is not in the idle band \[0.12·ω\_nominal, 0.35·ω\_nominal\] — exactly the real STCW check before engaging the propeller.

Excessive slip alarm (clutch\_slip): triggered if the clutch is fully engaged (position ≥ 0.99) AND |slip| \> 0.3·ω\_nominal, sustained ≥ 15 consecutive seconds.

### 2.5 Propeller shaft equation

eq\_hsg \= (torque\_hsg − friction\_hsg · ω\_ax) / gear\_ratio\_hsg

propeller\_resistance \= K\_p · ω\_ax · |ω\_ax|  (quadratic, as in a real propeller)

d(ω\_ax)/dt \= (clutch\_torque \+ eq\_hsg − propeller\_resistance) / J\_propeller

J\_propeller (propeller\_inertia, default 6000\) \>\> J\_motor — hence the large inertia during ship acceleration/deceleration.

## 3\. Operating modes and load distribution (PTO / MEC / PTI)

GEN:   P\_disp\_me \= P\_motor − hotel\_load   P\_disp\_el \= 0

MOTOR: P\_disp\_me \= P\_motor                P\_disp\_el \= P\_electrical − hotel\_load

OFF:   P\_disp\_me \= P\_motor                P\_disp\_el \= 0

Load distribution across sources depends on the mode — see MachineryMode.distribute\_load() for the exact three branches.

## 4\. Specific fuel consumption

Classical quadratic formula with real coefficients from the NTNU source:

specific\_consumption(load\_perc) \= max(0, a·load\_perc² \+ b·load\_perc \+ c) / 3.6×10⁹  \[kg/s per W\]

load\_perc is saturated at \[0, 1.2\] before calculation. Total fuel flow: fuel\_rate \= load\_me · specific\_consum(load\_perc\_me), accumulated over time for total session consumption.

## 5\. Starting air system

* Tank with initial pressure 30 bar (AIR\_MAX\_BAR).

* Engine start is rejected if pressure is below 18 bar (AIR\_MIN\_START\_BAR).

* A successful start consumes 6 bar (AIR\_CONSUMED\_PER\_START\_BAR).

* Refill: each powered air compressor adds 0.5 bar/s (COMPRESSOR\_FILL\_RATE\_BAR\_S); two compressors fill twice as fast.

## 6\. Cooling and oil pressure

The cooling model is two-stage (seawater → heat exchanger → freshwater → engine):

cooling\_frac \= 1.0  if (SW active) AND (FW active)

             \= 0.5  if only one of the two is active

             \= 0.0  if neither

target\_normal \= 65 \+ 20 · min(1, ω\_ax / ω\_nominal)

target \= target\_normal \+ (160 − target\_normal) · (1 − cooling\_frac)

Actual temperature relaxes exponentially towards the target (first-order discrete form):

T(t+dt) \= T(t) \+ (target − T(t)) · 0.05

This relaxation pattern (x \+= (target − x) · rate) is used throughout the model for any "slow" quantity (temperatures, tank levels).

Oil pressure with asymmetric rise vs. fall rates:

if pump running: P(t+dt) \= P(t) \+ (4.2 − P(t)) · 0.10

otherwise:       P(t+dt) \= P(t) \+ (0 − P(t)) · 0.20

## 7\. Diesel generators and power bus

### 7.1 Local generator governor

start\_ramp(t) \= sat(t / 8.0, 0, 1\)

droop \= (droop\_rpm\_full\_load · π/30) · load\_perc  (if breaker closed)

target\_ω \= ω\_nominal · ramp − droop

ω(t+dt) \= ω(t) \+ (target\_ω − ω(t)) · min(1, 6·dt)

droop\_rpm\_at\_full\_load \= 20 rpm — typical droop governor speed drop at 100% load.

### 7.2 Synchronisation and paralleling

close\_breaker() only succeeds if the generator is at rated speed within ±3% (at\_rated\_speed()). Otherwise, sync\_fault is declared — the classic STCW "generator paralleling" training scenario.

### 7.3 Load sharing

share\_i \= demand · (P\_nominal\_i / ΣP\_nominal\_online)

If demand exceeds total online capacity → overload \= True. If no generator is online and there is demand → blackout \= True: bus loses all power, and main engine electric pumps fail regardless of their local switch.

## 8\. Phase 1 — auxiliary systems and STCW faults

All quantities below use the same exponential relaxation pattern x \+= (target − x) · rate (section 6), or simple linear integration x \+= rate · dt for tank levels.

### 8.1 Thrust bearing

load\_frac \= min(1, ω\_ax / ω\_nominal)

target \= 55 \+ 25 · load\_frac

       \+ 25  if lubrication missing AND engine running

       \+ 35  if explicit fault injected (thrust\_bearing\_fault)

T(t+dt) \= T(t) \+ (target − T(t)) · 0.03

Alarm: T \> 85 °C.

### 8.2 Cylinder misfire and exhaust temperatures

baseline \= 20 \+ 280 · min(1, |throttle|)  (if engine running, else 20\)

target\_i \= baseline · 0.35  if i \== faulty cylinder

         \= baseline · 1.08  if there IS a faulty cylinder but i ≠ it

         \= baseline          if no cylinder is faulty

exhaust\_i(t+dt) \= exhaust\_i(t) \+ (target\_i − exhaust\_i(t)) · 0.08

Alarm: misfire\_detected \= engine running AND a misfire cylinder is set.

### 8.3 Turbocharger surging

Requires simultaneously: an injected fault (clogged air filter) AND sudden load variation at high speed:

d\_throttle \= |throttle(t) − throttle(t−dt)|

surge\_now \= turbo\_fault AND engine running AND |throttle| \> 0.55 AND (d\_throttle/dt) \> 0.05

if surge\_now: surge\_timer \= 6.0

else if surge\_timer \> 0: surge\_timer −= dt

Alarm: surge\_timer \> 0 (lasts 6 seconds from last trigger).

### 8.4 Fire in the separator compartment

Binary state with extinction conditional on both corrective actions:

if fire\_active:

    if QCV\_closed AND CO2\_released:

        extinguishing\_timer \+= dt

        if extinguishing\_timer \>= 3.0: fire\_active \= False; separator → FAULT

    else:

        extinguishing\_timer \= 0  (any missing action → timer resets)

This strict AND (not OR) is intentionally pedagogical: a single measure (only fuel isolation OR only CO₂) is insufficient in a real compartment fire.

### 8.5 Sewage, bilge, fuel service tank, incinerator

All four use simple linear integration level \+= (fill − empty) · dt, saturated at \[0, 100\]%.

MARPOL violation (marpol\_violation): triggered if sewage level remains above the sustained threshold for more than 30 seconds (\_sewage\_high\_timer \> 30\) OR incinerator is over-temperature — a brief exceedance promptly corrected does not constitute a real discharge violation.

### 8.6 The forced alarm mechanism (forced\_alarms)

All alarms calculated from real physics pass through a common filter (see §5 in the Developer Guide). A scenario can force an alarm (force\_alarm(key)) for procedural training. While the key is in forced\_alarms, the physical calculation in step() does not override it.

## 9\. Phase 6 — Extended physics

Phase 6 is implemented in engine\_core.py. These subsystems run every simulation tick alongside Phase 1 physics.

### 9.1 Dual HT/LT freshwater cooling circuits

Two separate freshwater circuits complement the base cooling model:

* fw\_ht (high-temperature, jacket water cooling) — feeds engine cylinder cooling.

* fw\_lt (low-temperature, central heat exchanger) — cools the HT return and lubricating oil.

lt\_target \= 38.0  if (sw\_ok AND fw\_lt\_ok)

          \= 55.0  if only one active

          \= 85.0  if neither

The HT circuit raises cooling\_water\_temp further if fw\_ht is inactive.

### 9.2 Fuel viscosity and preheater

if heater\_ok (fuel\_heater aux \+ bus powered \+ steam\_pressure \>= 2.0 bar):

    fuel\_viscosity \= 12.0 \+ max(0, (130.0 − fuel\_temp) · 0.35)

else:

    fuel\_viscosity \= 3.5  (cold, low viscosity HFO approximation)

Alarm high\_fuel\_viscosity: engine running AND viscosity \> 20.0 cSt.

### 9.3 Steam boiler

burner\_steam\_gen \= 0.04  if (boiler\_active AND bus\_powered AND boiler\_water\_level \> 15%)

                 \= 0.0   otherwise

boiler\_water\_level \+= (water\_refill − water\_evaporation) · dt  \[%, clipped 0–100\]

Alarms: boiler\_low\_water\_level (level \< 20%), boiler\_high\_pressure (steam\_pressure \> 8.5 bar).

### 9.4 Turbogenerator

turbogenerator\_running \= (steam\_pressure \>= 5.0)

turbogenerator\_power\_kw \= 120.0  if running  else 0.0

### 9.5 MARPOL Annex VI emissions

fuel\_rate\_kg\_h \= fuel\_rate\_kg\_s · 3600

emissions\_co2\_kg\_h \= fuel\_rate\_kg\_h · 3.114

sox\_raw\_kg\_h \= fuel\_rate\_kg\_h · (2.0 · fuel\_sulfur\_pct / 100.0)

emissions\_sox\_kg\_h \= sox\_raw\_kg\_h · (0.10  if scrubber\_on  else 1.0)

emissions\_nox\_g\_kwh \= 14.5  if (running AND NOT scr\_on)

                    \= 2.8   if (running AND scr\_on)

                    \= 0.0   if not running

alarm emissions\_violation: running AND (SOx \> 2.5 kg/h OR NOx \> 10.0 g/kWh)

## 10\. Scenario Engine — fault injection signal shapes (Phase 4\)

A continuous\_trigger modifies a state variable in real time, according to 6 possible mathematical forms (t \= time elapsed since trigger). The base\_value is automatically captured from the current state on the first tick if not explicitly specified.

| Shape | Formula | Notes |
| :---- | :---- | :---- |
| ramp | v(t) \= start \+ (end−start) · t/duration | Remains at end value after expiry |
| step | v(t) \= initial (t \< delay), step\_value (t ≥ delay) | Instantaneous switch |
| pulse | v(t) \= pulse\_val (t \< width), base (t ≥ width) | Single pulse then returns to base |
| pulse\_train | v(t) \= pulse\_val (t mod period \< width), else base | Repetitive pulses |
| sine | v(t) \= base \+ amplitude · sin(2π · freq · t) | Continuous oscillation |
| random | v(t) \= base \+ noise (uniform); Random Walk: accumulates deviation | Noise simulation |

## 11\. Evaluation engine — scoring formula

For each non-penalty criterion with score p, if satisfied, add p to total\_awarded; the maximum possible is the sum of all positive scores, total\_max.

final\_score \= clamp( round( 100 · total\_awarded / total\_max ), 0, 100 )

Penalty criteria subtract directly from total\_awarded (do not contribute to total\_max) — a single dangerous action can take the score below the raw positive total.

STCW segmented report: each criterion is labelled with an STCW category; for each category:

category\_pct \= clamp( 100 · (awarded\_category − deductions\_category) / max\_category, 0, 100 )

"Completed" exercise: final\_score ≥ 70 AND no penalty applied.

## 12\. e-Coach — message trigger mechanism

Each rule (CoachRule) has a periodically evaluated Boolean condition, plus two temporal parameters:

* trigger\_delay\_s: the condition must remain true continuously for this duration before the message is emitted (avoids false alarms on fraction-of-a-second transients).

* cooldown\_s: after issuing, the rule cannot re-trigger for this duration even if the condition remains true.

if condition(state) \== True:

    if condition\_true\_since \== None: condition\_true\_since \= now

    if (now − condition\_true\_since) \>= trigger\_delay\_s

    AND (now − last\_triggered\_at) \>= cooldown\_s:

        emit message; last\_triggered\_at \= now

else:

    condition\_true\_since \= None

## 13\. Known model limitations

This simulator is an educational platform, not a commercial class simulator (Kongsberg/Wärtsilä). Consciously assumed simplifications:

* Tank temperatures and levels use first-order exponential relaxation / linear integration — not full thermodynamic equations (no real heat capacity, no surface heat transfer model).

* There are no pressure wave patterns in fuel/air/oil circuits — pressures are scalar states with first-order dynamics.

* Propeller resistance is a simple quadratic model (K·ω·|ω|), without cavitation effects, non-uniform flow, or hull interaction.

* Turbogenerator power is computed and offsets the electrical bus demand in parallel with the diesel generators.

* Phase 1 models for tanks (sewage/bilge/fuel/incinerator) are linear rates/first-order relaxations, sufficient for STCW training but not full thermodynamics.

* e-Coach rules and exercise texts are in Romanian only, regardless of UI language.