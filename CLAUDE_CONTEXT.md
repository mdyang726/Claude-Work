# Project Context for Claude — Autonomous Propulsive Landing Model Rocket

*Read this entire file before helping with anything. It is the authoritative context for this project.*

---

## What This Project Is

A single-stage model rocket that launches, autonomously detects flight conditions via onboard sensors and a 5-state machine, and executes a propulsive vertical landing — no human intervention after ignition. Mirrors SpaceX Falcon 9 booster recovery at hobby scale.

**Builder profile:** High school / early college, strong C++, 3D printer access, flexible budget, rapid timeline (started June 5, 2026).

**Target outcome for summer:** At least one documented successful propulsive vertical landing on video.

**Why it matters:** Aerospace portfolio piece. A landing video proves GNC loop closure on real hardware.

---

## Hardware Stack (Locked)

| Component | Part |
|-----------|------|
| Flight computer | Teensy 4.0 (600MHz Cortex-M7) |
| IMU | TBD (must support 100–200Hz; used for attitude estimation + double integration altitude) |
| GPS | Optional — provides absolute altitude cross-check if onboard the Teensy module |
| Ascent motor | Estes F-class, shared TVC gimbal (servo + PID) |
| Landing motor | Estes F-class, 0-sec delay, e-match triggered — occupies same gimbal after launch motor ejects |
| Landing legs | 4× spring-loaded, deployed by flight computer |

---

## Flight Profile

```
Launch → Boost (TVC active, PID attitude control, launch motor)
       → Coast (ascending, TVC off, launch motor spent and ejected)
       → Apogee (velocity = 0, detected via IMU)
       → Descent (low-CoM passive stability, landing motor pre-staged in gimbal)
       → Ignition trigger (IMU double integration altitude hits threshold)
       → Landing burn (landing motor fires in same gimbal, same PID code, legs deploy)
       → Touchdown (legs absorb impact)
```

## 5-State Machine

1. **Boost** — motor burning, TVC active, watching for burnout
2. **Coast** — ascending passively, waiting for apogee (v=0 via IMU)
3. **Descent** — computing ignition altitude from measured apogee, watching for threshold
4. **Ignition** — landing motor fired, legs deploying
5. **Landing** — post-touchdown safe state (impact via IMU)

---

## The Most Critical Engineering Fact

> **Effective ignition control window: ~200–400ms before latency subtraction.**

At 30–50ft apogee with ~12 m/s descent, time-to-impact at 3m detection altitude is roughly 250ms. Subtract:

| Latency source | Conservative estimate |
|---------------|----------------------|
| IMU read + integration cycle (200Hz) | 5ms |
| Teensy loop cycle (100–200Hz) | 5–10ms |
| MOSFET switching | 1ms |
| E-match ignition | 50ms |
| Motor spool to meaningful thrust | 50ms |
| **Total** | **~111–116ms** |

Residual window = ~119ms at nominal conditions. This is tight. Architecture is viable but leaves little margin for surprises. All latency terms must be characterized empirically before flight.

**If residual < 100ms after running the simulation with actual vehicle parameters → architecture must change** (higher detection altitude, higher apogee, drogue chute, or faster sensor stack).

---

## Key Technical Constraints (Non-Negotiables)

**Altitude via IMU double integration.** Integrate vertical acceleration → velocity, integrate again → altitude. Works well over the short terminal phase window (drift is negligible over <2 seconds). Reset the integrator to zero at apogee (a known zero-velocity event) to bound drift over the full flight. GPS onboard gives a free absolute-altitude cross-check if the Teensy module includes it.

**Single gimbal, shared TVC code.** The launch motor ejects at burnout. The landing motor occupies the same gimbal mount. The same PID attitude control block runs both burns — TVC is actively correcting attitude throughout the landing burn, not just during ascent. During the passive descent phase (no motor burning), TVC is inactive and low-CoM passive stability handles attitude.

**Passive descent stability still required.** From burnout through ignition, there is no active control. The vehicle must arrive at the ignition altitude sufficiently upright for TVC to take over. Low CoM is the correct design for this phase. Quantify the acceptable off-axis envelope and encode it as a hardcoded abort threshold.

**Abort thresholds hardcoded, never field-tuned.** Max attitude error at ignition point, max altitude deviation, min/max ignition window — all decided before launch day and committed into the state machine. If any threshold is violated, state machine aborts to safe state (no ignition).

**Log everything.** Every sensor reading, state transition, computed value at maximum storage rate. A crash with logs is a design input. A crash without logs is wasted hardware.

**Early-ignition bias.** The control law should bias toward early ignition. Motor fires slightly early → gentle landing from thrust tail-off. Motor fires slightly late → ground impact at speed. Asymmetric consequences → bias toward early.

---

## 10-Week Timeline (Reference)

| Weeks | Work |
|-------|------|
| 1–2 | **Terminal phase simulation script** + airframe CAD + gimbal design begins |
| 3–4 | TVC static fire + PID tuning on test stand |
| 5–6 | Avionics integration + sensor fusion + HIL state machine testing |
| 7 | Leg deployment bench tests + dual e-match timing verification |
| 8 | **Min viable flight:** TVC disabled, passive stability only. Validate state machine + ignition trigger. |
| 9–10 | Full system flight attempts. Analyze logs after each. Iterate. |

**Critical path:** TVC gimbal + PID tuning. Must not slip past week 6 or full-system flights lose margin. Start gimbal mechanical build no later than week 2.

---

## Regulatory (Must Do Before Hardware Complete)

- NAR Level 1 certification (likely required for F-class motors)
- Club pre-approval for powered descent flight profile — contact RSO before arriving at range
- TVC system explicit RSO sign-off at most NAR/Tripoli ranges
- Pyrotechnic handling certification if required by local club
- Pre-define failure mode: if landing motor fails to ignite, vehicle descends and impacts ground — safety perimeter must be defined for this outcome

---

## Leg Design Spec

Design to the **5 m/s touchdown** case, not nominal (2 m/s).

| Touchdown velocity | Force per leg (4 legs, 0.5kg, Δt=50ms) | Note |
|---|---|---|
| 2 m/s (nominal) | ~20N | Fine with PLA |
| 5 m/s (design spec) | ~50N | Use PETG, reinforce cross-sections |
| 10 m/s (motor misfire) | ~100N | Accepted write-off |

---

## Current Project Status (as of June 5, 2026)

**What has been done:**
- Project fully scoped through two rounds of LLM Council analysis
- Architecture locked (see hardware stack above)
- All major design constraints and go/no-go gates identified
- Detailed simulation guide written — ready to execute

**What has NOT been done yet:**
- Terminal phase simulation script has not been run (this is step 1)
- No hardware has been ordered
- No OpenRocket model exists yet
- No physical build has started

**Where to start:** Run the terminal phase simulation first. Everything else depends on it.

---

## Files in This Project Folder

| File | What it is |
|------|-----------|
| `PROJECT_BRIEF.md` | Full design brief from LLM Council analysis. Read this for deep context on every subsystem. |
| `council-report-20260605.html` | HTML report from the first council run. Open in a browser. Covers the sequencing question. |
| `terminal-phase-sim-guide.md` | **Start here for active work.** Step-by-step guide to writing and running the terminal phase simulation in Python — includes the full script, all design decisions, latency budget, sensitivity analysis, and research checklist. |
| `CLAUDE_CONTEXT.md` | This file. |

---

## How to Help With This Project

**Always read `PROJECT_BRIEF.md` if you haven't.** It contains deep subsystem-level detail this summary omits.

**The immediate next task is the terminal phase simulation.** `terminal-phase-sim-guide.md` has everything needed to run it. The user may need help: setting up Python, understanding the physics, filling in parameter values, interpreting the output, or troubleshooting the script.

**Key things to watch for:**
- If the user asks about motor selection, point them to thrustcurve.org and their specific Estes motor part number
- If residual window comes out < 100ms in the simulation, flag this prominently — it means architecture changes, not just tuning (higher apogee is the first lever to pull since it directly expands the window)
- The TVC gimbal is the critical path. Any conversation about schedule should treat it as the item that cannot slip
- Never encourage skipping the simulation to go straight to building. The simulation is a feasibility gate, not optional groundwork

**Preferred communication style:** Concise and direct. Minimize unnecessary explanation. The user is technically capable — treat them as a peer, not a beginner.
