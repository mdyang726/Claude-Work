# Autonomous Propulsive Landing Model Rocket
### Project Brief & Design Insights
*Generated June 5, 2026 — synthesized from LLM Council analysis (v1 + v2)*

---

## What This Project Is

You are building a single-stage model rocket that launches, autonomously detects flight conditions via onboard sensors and a state machine, and executes a propulsive vertical landing — all without human intervention after ignition. The concept mirrors SpaceX Falcon 9 booster recovery in architecture, scaled to hobby rocketry constraints.

This is not a simulation project. It is a real flight vehicle that will either land or it won't, and the reason it matters as a portfolio piece is exactly that: the outcome is unambiguous and the engineering required to get there is broad and deep. A successful landing video is worth more to an aerospace employer than a semester of MATLAB sims, because it proves you can close a GNC loop on real hardware under real constraints.

**Target outcome for summer:** At least one documented successful propulsive vertical landing. Reliable, repeatable landing is a follow-on season goal — don't let the perfect be the enemy of the good.

---

## System Architecture

```
FLIGHT PROFILE
Launch → Boost (ascent motor, TVC active) → Coast → Apogee
→ Descent (low-CoM passive stability) → Ignition (landing motor)
→ Landing burn → Touchdown (legs absorb impact)

HARDWARE
┌─────────────────────────────────────────────┐
│ Flight Computer: Teensy 4.0 (600MHz M7)     │
│ Sensors: Barometer/altimeter + IMU          │
│ State machine: 5 phases                     │
└─────────┬───────────────────────────────────┘
          │
    ┌─────┴──────────────────────────────────┐
    │  ASCENT               DESCENT/LANDING  │
    │  Estes F-class        Estes F-class    │
    │  TVC gimbal           0-sec delay      │
    │  Servo + PID          E-match trigger  │
    │  Vertical             Low CoM passive  │
    │  orientation          orientation      │
    └────────────────────────────────────────┘
          │
    Spring-loaded legs (4x), deployed by flight computer
    Linear control law: apogee altitude → ignition altitude
```

**Builder profile:** High school / early college, strong C++, 3D printer access, $500 budget, ~10 weeks.

---

## The Number That Defines Everything

> **Effective ignition control window: 200–400ms — before latency.**

This is the single most important engineering fact about this project. Work it out for your specific vehicle before you write a line of flight code.

At 30–50 ft apogee with drag, your descent velocity at the landing motor ignition point will be approximately **8–14 m/s**. If ignition fires at roughly 3m altitude (10 ft) with 12 m/s descent, time-to-impact is about 0.25 seconds — your gross window. From that, subtract:

- E-match ignition latency: **~50ms**
- Sensor read cycle + compute: **~20–50ms** depending on your loop rate
- Any additional relay/logic switching: **~10–20ms**

What's left is your actual margin. **If it's under 100ms, your architecture needs to change** (higher target altitude, different motor selection, or adding a drogue chute to reduce descent velocity). If it's 150–200ms, you can proceed but abort logic is mandatory. Over 200ms, you have room to work.

**Do this calculation in a script before anything else.** Inputs: vehicle mass, drag coefficient (estimate from geometry), motor thrust curve, apogee altitude. Output: descent velocity as a function of altitude, ignition window in milliseconds after subtracting your specific latency budget. This script is the specification for the entire avionics subsystem.

---

## Key Design Insights by Subsystem

### 1. Propulsion — Motor Selection & Characterization

**The control law assumption:** The linear apogee-to-ignition-altitude mapping assumes a deterministic, consistent descent trajectory. It works because physics is predictable *if* the motor behaves predictably.

**What to watch:**
- Estes F-class motors have batch-to-batch impulse variance of roughly ±15%. This matters for two reasons: (1) it introduces scatter in your landing burn velocity, and (2) your legs must be designed for worst-case impulse, not nominal.
- Buy several motors from the same production batch if possible. Static-fire at least two before your first flight to characterize the actual thrust curve and measure ignition-to-thrust-onset latency with your specific e-match and relay configuration.
- The "0-second delay" designation refers to the ejection charge delay, not ignition latency. Actual e-match ignition delay in your configuration needs to be measured empirically, not assumed.
- Your control law should have an **early-ignition bias** baked in. A landing motor that fires slightly early produces a gentle return from the tail-off. A motor that fires slightly late produces ground impact at speed. Asymmetric consequences → bias toward early.

**The dual-motor ignition offset problem (critical, easy to miss):**
You have two motors with two separate e-matches and two separate pyro channels. Even a 20–50ms offset between the two igniting produces asymmetric thrust during the most critical phase of flight — when dynamic pressure is near-zero and TVC has no aerodynamic authority. Bench-verify that your dual ignition circuit fires within 10ms before any flight. This is a hard go/no-go gate, not a nice-to-have.

---

### 2. Guidance & Sensing — Sensor Architecture

**The pure-barometer problem:**
A BMP388-class barometric altimeter runs at ~25Hz with 0.5–1m altitude noise. At 30–50 ft altitude with a descent phase under 2 seconds, this is marginal. The sensor update period (40ms) alone is a meaningful fraction of your available timing window. Pure barometer is not sufficient for the terminal phase.

**The required fix — sensor fusion:**
Fuse barometric altitude with IMU accelerometer velocity integration for the terminal descent phase. The barometer gives you absolute altitude reference (low noise over long time, high noise over short time). The accelerometer integrates to velocity (accurate over the short terminal window, drifts over long periods). Together they give you the continuous, low-latency altitude and velocity estimate you need to compute the ignition moment. This is not optional at 30–50 ft target altitude.

**IMU considerations:**
- Use the IMU for both attitude estimation (during boost and descent) and velocity integration (terminal phase).
- Your PID loop for TVC needs the attitude data at a loop rate fast enough to be useful during a 1–2 second burn. 100–200Hz is typical for hobby flight controllers; make sure your Teensy loop is structured to support this.
- Log everything. Every sensor reading, every state transition, every computed value. Treat every flight as a data collection event, not just a pass/fail test.

---

### 3. Attitude Control — The Two Hardest Moments

There are two distinct attitude control challenges in this flight profile, and they operate on completely different physics. Don't conflate them.

**Hardest Moment #1: Ascent burn (TVC active)**
TVC is your active control during boost. The Teensy reads IMU attitude and drives servo corrections through a PID loop. This is the subsystem with the most iteration required — expect to tune PID gains through multiple static fire sessions before flying free. The 3D printer is your best friend here; iterate the gimbal mount geometry aggressively.

Key points:
- Tune on a static thrust stand (motor mounted horizontal, vehicle restrained, free to rotate in pitch/yaw). You can close the PID loop and observe oscillation, overshoot, and response time without flying.
- Start with conservative (low) proportional gains and work up. Oscillation during burn will destabilize faster than you can correct manually.
- The center of pressure moves during burn as the motor consumes propellant and vehicle CoM shifts. Account for this in your stability analysis.

**Hardest Moment #2: Landing burn ignition transient (the one nobody talks about)**
At the moment the landing motor ignites, airspeed is approximately zero. TVC has zero aerodynamic authority for the first 100–200ms of the burn while thrust is building and dynamic pressure remains low. This means: **the vehicle must already be attitude-stable before ignition.** If it arrives at the ignition point 15 degrees off-axis — even with TVC active on the landing motor — you will get a lateral crash, not a landing.

This is what makes the passive descent stability design so important. Low CoM is the right call for scope reasons, but you need to understand its limitations:
- The "pendulum effect" produces a weak restoring torque at low airspeeds. At 10 m/s descent with modest lateral velocity from apogee, the aerodynamic restoring force is small.
- Any lateral velocity component acquired during or after apogee (tumbling, wind shear) persists through descent. There is no self-correcting force strong enough to zero it before ignition.
- **Define a quantified acceptance criterion for off-axis angle at ignition.** Something like: "If attitude error exceeds X degrees at [altitude trigger], abort ignition." This goes in the state machine as a hardcoded threshold, not a field judgment call.

**Practical implication:** Build and validate the passive descent stability early — drop tests, wind-tunnel-equivalent bench tests — before relying on it in flight. Know your actual off-axis envelope before you fly.

---

### 4. Landing Legs — Design to Failure Modes

Legs are straightforward mechanically, but the load cases span a wide range depending on how well the burn went.

| Touchdown velocity | Force per leg (4 legs, m=0.5kg, Δt=50ms) | PLA viability |
|---|---|---|
| 2 m/s (nominal landing) | ~20 N | Fine |
| 5 m/s (partial burn / early cutout) | ~50 N | At risk — reinforce cross-sections or use PETG |
| 10 m/s (motor misfire) | ~100 N | Certain structural failure — accept write-off |

**Design to the 5 m/s case.** The 2 m/s case is what you're aiming for; the 5 m/s case is what you should design for; the 10 m/s case is a total vehicle write-off and that's an acceptable pre-accepted failure mode.

Additional considerations:
- Validate leg deployment with drop tests before flying. Drop the vehicle from the target leg-deployed height onto a hard surface and verify the mechanism releases correctly under G-load, then absorbs impact without structural failure.
- Leg spring rate determines both release force and impact absorption. Tune them together — a spring stiff enough to hold the leg stowed must be compliant enough to absorb landing. These are often in tension.
- Wire-melt or servo release reliability: test the release mechanism under the temperature and vibration conditions of flight (the burn will heat components). A leg that fails to deploy turns a landing into a no-leg crash.

---

### 5. Flight Software — State Machine Design

The state machine is the brain. Get it right before flying.

**Five states:**
1. **Boost** — motor burning, TVC active, watching for burnout (thrust termination via altimeter + IMU)
2. **Coast** — ascending passively, waiting for apogee detection (velocity = 0 via IMU)
3. **Descent** — falling, computing ignition altitude from measured apogee, watching for ignition threshold
4. **Ignition** — landing motor fired, legs deploying, TVC may be active if motor is gimbaled
5. **Landing** — post-touchdown detection (impact via IMU), safe state

**Non-negotiables in the code:**
- **Abort thresholds hardcoded, not tuned in the field.** Before any flight day, define and commit to numerical go/no-go criteria: max attitude error at ignition, max altitude deviation from predicted trajectory, min/max ignition window. If any threshold is violated, the state machine aborts to a safe state (no ignition, vehicle descends and hits the ground on legs if deployed, or on motor casing if not).
- **Blackbox logging on every flight.** Log timestamps, sensor readings, state transitions, computed values, and command outputs at the highest rate your storage allows. A crash with full logs is a learning event. A crash without logs is wasted hardware.
- **Hardware-in-the-loop testing before flying.** Feed simulated sensor data into the Teensy and verify every state transition, every abort condition, and every edge case before the vehicle ever leaves the ground. You have very few flight opportunities — don't spend one debugging a state machine bug.

---

### 6. Budget & Timeline Reality

**Budget ($500):**

| Item | Estimate |
|---|---|
| 2× Estes F motors + reloads | ~$60 |
| Teensy 4.0 + BMP388 + IMU | ~$80 |
| Servos, gimbal hardware, filament | ~$80 |
| Airframe tube, rings, fins | ~$60 |
| Pyro / ignition components | ~$40 |
| Legs, springs, hardware | ~$40 |
| Contingency | ~$140 |
| **Total** | **~$500** |

The 3D printer is your force multiplier — it turns $40 of filament into rapid iteration on the most mechanically complex subsystems (gimbal mount, leg mechanism, airframe adapters). Use it aggressively.

**10-week timeline:**

| Weeks | Work |
|---|---|
| 1–2 | Terminal phase simulation script. Airframe design + CAD. Begin gimbal mount design. |
| 3–4 | TVC static fire + PID tuning. Iterate gimbal geometry. |
| 5–6 | Avionics integration. Sensor fusion implementation. State machine HIL testing. |
| 7 | Leg deployment bench tests. Ignition circuit bench verification (dual e-match timing). |
| 8 | **Minimum viable flight:** passive stability only (no TVC), validate state machine + ignition trigger. Log and analyze. |
| 9–10 | **Full system flight attempts.** Analyze logs after each flight. Iterate. |

**Critical path:** TVC gimbal + PID tuning. If this slips past week 6, you have no margin for full-system flights. This is the one subsystem where bench testing is hardest to substitute for real iteration — start it as early as possible.

**Each flight is expensive.** Unlike software iteration, each flight costs a motor, risks hardware damage, and takes a day to set up. Maximize the number of subsystems you can test on the bench without flying. More bench tests per flight = more useful flights.

---

## Testing Philosophy

The most important meta-insight from the council: **treat every test as a data collection event, not a pass/fail event.**

A flight where the landing motor fires at the right moment but the rocket tips over is a near-success that tells you your descent attitude wasn't stable enough at ignition. A flight where the motor fires too late tells you your timing model needs recalibration. A flight where the motor doesn't fire at all tells you your ignition circuit has a reliability problem. All of these are useful — but only if you logged enough data to diagnose them.

**Testing sequence by subsystem (bench-testable, no flight needed):**

- **State machine:** Hardware-in-the-loop. Feed simulated sensor inputs to the Teensy. Walk through every state, every abort condition, every edge case. Do this until it's boring.
- **Barometer + IMU:** Install in vehicle, log data during a car ride over a bridge or parking structure to validate altitude detection. Log data during manual rotation to validate attitude estimation.
- **Leg deployment:** Suspend the vehicle at leg-deploy altitude, trigger via flight computer, measure time-to-deploy under G-load. Then drop from 6 feet with legs deployed, verify spring travel and structural integrity.
- **Ignition circuit:** Fire into a resistor load (not a real motor). Measure relay switching time. Measure dual e-match firing offset. If offset is >10ms, debug and fix before flying.
- **TVC:** Mount motor horizontal in a static thrust stand. Run PID loop live while motor burns. Tune gains, measure oscillation frequency, verify burn-end behavior.

**First flight (Week 8 — minimum viable):** Disable TVC. Fly with passive stability only. This flight's purpose is to validate: (1) the state machine transitions correctly in a real flight environment, (2) the barometric ignition trigger fires at the right altitude, and (3) the vehicle survives boost and descent in a stable attitude. Don't try to land it on flight 1 — use parachute or accept the hard landing. The data from this flight calibrates your model for full-system flights.

---

## Regulatory Checklist

Do this before hardware is complete — approval timelines can be weeks.

- [ ] **NAR Level 1 certification** — likely required for F-class motors. Check your specific motor's certification requirement and your launch site's rules.
- [ ] **Club pre-approval for non-standard flight profile** — most NAR/Tripoli clubs require Range Safety Officer review for powered descent. Contact your local club and describe the flight profile before showing up. Bring documentation.
- [ ] **TVC system approval** — gimbaled thrust during ascent may require explicit RSO sign-off at many ranges. Find out in advance.
- [ ] **Pyrotechnic handling certification** — if your club requires it for onboard ignition systems beyond the standard launch controller setup.
- [ ] **Pre-define failure mode consequence** — NAR range safety will want to know what happens if the landing motor fails to ignite. Answer: vehicle descends and impacts the ground. Pre-commit to a safety perimeter appropriate for this outcome.

---

## The Prior Existence Proof — What It Tells You and What It Doesn't

A hobbyist has already demonstrated propulsive landing using this general architecture (barometric state machine, timed ignition, passive descent stability). This is meaningful — it validates that the physics and the basic control approach work. You are not trying to prove an unknown concept.

What the existence proof does **not** tell you:
- What target altitude that hobbyist used (almost certainly higher than 30–50 ft, where the timing window is proportionally larger and barometric noise is proportionally less critical).
- Whether that landing was reliable and repeatable, or a one-time success on the right day with the right motor.
- Whether their sensor architecture, motor selection, and control law parameters are directly applicable to your vehicle.

Use it as existence proof of concept, not as a recipe. Build your own simulation from your own vehicle parameters, and derive your own ignition altitude from that — don't copy a number from someone else's design.

---

## One-Sentence Summary of Each Council Insight

- **The timing window is your specification.** Compute it before you design anything else.
- **Low CoM is correct but incomplete.** It reduces the problem; it doesn't eliminate descent attitude risk. Quantify the acceptable off-axis envelope.
- **Sensor fusion is required.** Pure barometer at this altitude and time scale is insufficient for the terminal phase. Fuse with IMU.
- **Dual e-match synchronization is a go/no-go gate.** Bench-verify to <10ms before any flight.
- **TVC has no authority at ignition.** The vehicle must arrive at the ignition point already stable. Your descent stability system is what makes or breaks the landing, not TVC.
- **Design legs for the 5 m/s case.** Nominal is 2 m/s; design spec is 5 m/s; 10 m/s is an accepted write-off.
- **Hardcode abort thresholds.** Decide before field day what numbers cause a no-ignition abort. Don't make this call on the launch pad.
- **Log everything.** Every flight is worth more with data than without. A crash with logs is a design input. A crash without logs is a write-off.
- **NAR/Tripoli approval first.** The regulatory timeline starts when you submit. Contact your club before hardware is finalized.
- **Motor characterization before flight.** Static-fire your batch, measure ignition latency, measure thrust curve variance. Design to worst-case, not nominal.

---

*This document was generated from a two-round LLM Council analysis (5 advisors × 5 peer reviewers × chairman synthesis) conducted June 5, 2026. Council reports: `council-report-20260605.html` (initial spec) and `council-report-v2-20260605.html` (full spec).*
