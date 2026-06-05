# Terminal Phase Simulation — Step-by-Step Guide

*How to write, run, and interpret the ignition window feasibility script*

---

## What This Script Actually Does

The simulation answers one question: **at the moment your sensor detects the ignition altitude, how much time remains before ground impact after subtracting every real-world delay between detection and meaningful thrust?**

If that number is positive with margin → proceed. If it's negative or near zero → your architecture must change before you order anything.

The simulation models:
1. Your rocket falling from apogee under gravity + aerodynamic drag
2. Velocity and altitude at every millisecond of descent
3. The moment your detection altitude is crossed
4. Subtraction of every real latency in your system
5. What's left over — the residual window

---

## Part 1: Where to Run This

### Option A — Local Python (Recommended)

This is the right long-term setup. You'll use Python throughout this project for data analysis, log parsing, and future simulations.

**Install:**
1. Download Python 3.11+ from [python.org](https://python.org). During install, check "Add Python to PATH."
2. Install VS Code from [code.visualstudio.com](https://code.visualstudio.com).
3. Install the Python extension inside VS Code (search "Python" in the Extensions panel).
4. Open a terminal in VS Code (`Ctrl+\``) and run:
   ```
   pip install numpy matplotlib
   ```
5. Create a file called `terminal_sim.py` and run it with the Play button or `python terminal_sim.py` in the terminal.

**Research if needed:** "Python VS Code setup tutorial" — Microsoft's own docs are the best starting point.

### Option B — Google Colab (No install, runs in browser)

Go to [colab.research.google.com](https://colab.research.google.com), create a new notebook, paste code into a cell, and press Shift+Enter to run. numpy and matplotlib are pre-installed. Good for quickly prototyping before committing to a local setup.

### Option C — Replit

Go to [replit.com](https://replit.com), create a Python repl, paste your code. Requires an account. Slightly slower than local but works from any machine.

---

## Part 2: The Physics Model

This is the core of the simulation. You need to understand each piece before you can make good assumptions.

### 2.1 Forces on the Rocket During Descent

During descent (after apogee, before landing motor ignition), two forces act on the rocket:

- **Gravity** pulls it down: `F_gravity = m × g`
- **Aerodynamic drag** resists motion (acts upward, opposing the downward velocity): `F_drag = 0.5 × ρ × v² × Cd × A`

Net downward force: `F_net = m×g − F_drag`

Net downward acceleration: `a = g − (ρ × v² × Cd × A) / (2 × m)`

Where:
- `m` = rocket mass in kilograms
- `g` = 9.81 m/s²
- `ρ` = air density (kg/m³) — 1.225 at sea level
- `v` = current velocity (m/s), starts at 0 at apogee, increases as it falls
- `Cd` = drag coefficient (dimensionless) — this is a design decision, covered in Part 3
- `A` = reference area (m²) — the cross-sectional area of your rocket body

### 2.2 Numerical Integration

You can't solve this analytically because drag depends on velocity which is changing. Instead, you step forward in small time increments (dt = 1ms works well):

```
At each time step:
  compute drag force from current velocity
  compute net acceleration
  update velocity: v_new = v + a × dt
  update altitude: h_new = h − v × dt   (altitude decreases as it falls)
  record (time, velocity, altitude)
  if altitude <= detection_altitude: record the detection moment
  if altitude <= 0: stop (ground impact)
```

This is Euler integration — simple, sufficient for this purpose. With dt=1ms and a ~2-second descent, you're doing ~2000 steps. Computationally instant.

### 2.3 Terminal Velocity

With drag included, the rocket doesn't accelerate forever — it approaches a terminal velocity where drag equals gravity. For a small rocket, terminal velocity during descent is typically 8–18 m/s depending on mass and Cd. This is what you're trying to nail down.

---

## Part 3: Vehicle Parameters — What You Need and How to Get Them

These are the inputs to your simulation. Each one requires a decision or measurement.

### 3.1 Mass at Landing (`m`)

This is the rocket's mass **at the moment the landing motor fires**, not at launch.

At landing, you have:
- Empty airframe + fins + electronics + legs
- Landing motor (full — it hasn't fired yet)
- **No** ascent motor propellant (it was burned on the way up)
- The ascent motor casing is still there (it doesn't eject)

**How to get it:** Weigh everything. If you don't have all the parts yet, estimate from component specs. Estes motor casings and their propellant masses are published in the motor's spec sheet. Add them up. A rough starting estimate for this vehicle is 400–600g.

**Why it matters:** Higher mass = more gravity force = faster terminal velocity = less time to impact. The simulation is highly sensitive to mass — be honest about it.

### 3.2 Drag Coefficient (`Cd`)

This is the hardest parameter to get right without wind tunnel data.

**What it is:** A dimensionless number that characterizes how much aerodynamic drag your shape produces. Cd=0 is theoretical (no drag). A blunt cylinder is ~1.0. A well-designed rocket nose cone + body is ~0.4–0.7 during unpowered flight.

**Research direction:** Look up "drag coefficient model rocket" on the NAR (National Association of Rocketry) website and in OpenRocket's documentation. OpenRocket actually computes Cd from geometry — you'll use this number later when you build the OpenRocket model. For now, bracket it: run your simulation at Cd=0.4 (optimistic), Cd=0.6 (nominal), and Cd=0.8 (pessimistic for descent with legs deployed).

**Critical note:** Your Cd changes between ascent and descent. During descent, you're likely falling nose-down or in an uncertain attitude (low CoM passive stability), legs may be partially deployed, and the flow field is different from ascent. The descent Cd is almost certainly higher than your ascent Cd. Use a conservative (higher) value.

**How it connects to the project brief:** The brief says "low CoM passive stability" produces uncertain attitude during descent — this directly increases your effective Cd.

### 3.3 Reference Area (`A`)

This is the cross-sectional area of your rocket's body tube — the area you'd see if you looked at the rocket from the side that the airflow hits.

`A = π × r²`

where `r` is the outer radius of your body tube in meters.

**Example:** A 54mm (Estes standard) body tube has r = 0.027m, so A = π × 0.027² ≈ 0.00229 m²

You need to decide on body tube diameter before running the sim — but this is usually constrained by your motor diameter anyway. Estes F-class motors require a 29mm motor mount, which fits inside a standard 54mm body tube.

### 3.4 Apogee Altitude (`h_apogee`)

This is where the simulation starts. At apogee, velocity = 0.

**How to get it (before OpenRocket):** Use a quick estimate. Estes F-class motors (specifically F15 or F39) deliver around 30–40 Newton-seconds of total impulse. For a 500g rocket with F15-0 motor, apogee will be somewhere in the 30–80ft range depending on drag and motor thrust curve. The project brief says 30–50ft is the target range.

**Start your simulation at 50ft (15.2m).** This is a conservative choice — lower apogee = faster descent at ignition altitude = less timing margin. If your design passes at 15m apogee, it passes at higher altitudes too.

**Once you have OpenRocket:** Replace this with the actual simulated apogee. But for the feasibility gate, start with a pessimistic estimate.

### 3.5 Detection Altitude (`h_detect`)

This is the altitude at which your barometer reads "fire now." It is a **design variable** — you choose it. Your control law maps measured apogee to this altitude.

**Starting point:** 3m (approximately 10ft) is the value from the project brief. This is where the ~250ms window comes from.

**What the simulation tells you:** At 3m altitude with your specific mass and Cd, what is the descent velocity? How many milliseconds remain until ground impact? That's your gross window before any latency subtraction.

**Design insight:** If the window is too tight, you have two levers: increase detection altitude (fire earlier, from higher up — but then descent velocity at ignition is lower, which means the burn may not zero out velocity before touchdown) or increase apogee target (more time to fall, slower terminal velocity). These tradeoffs are exactly what the simulation reveals.

---

## Part 4: The Latency Budget — Every Millisecond Accounted For

This is what separates a real feasibility analysis from a back-of-the-envelope guess. Stack these in order — each one subtracts from your gross window.

### 4.1 Barometer Sensor Update Period

The BMP388 (and similar Bosch sensors) uses oversampling to reduce noise. Higher oversampling = lower noise = higher latency. This is a design decision you control in firmware.

| Mode | Oversampling | Typical update period | Pressure noise (std dev) |
|------|-------------|----------------------|--------------------------|
| Fastest | OSR x1 | ~5ms | ~0.9 Pa (~7.5cm altitude noise) |
| Low noise | OSR x4 | ~20ms | ~0.5 Pa (~4cm noise) |
| Medium | OSR x8 | ~40ms | ~0.35 Pa |
| High | OSR x16 | ~80ms | ~0.25 Pa |
| Ultra | OSR x32 | ~160ms | ~0.18 Pa |

**Research direction:** Look up "BMP388 datasheet" — Section 3.4 covers output data rate and oversampling settings. The Bosch datasheet is freely available.

**The tradeoff:** At low altitude and high descent speed, you want fast updates. But fast updates mean noisy altitude readings — a reading with 7.5cm noise at 3m altitude has 2.5% relative error. A false trigger from noise firing the motor 200ms too early at terminal velocity could be as bad as a late trigger.

**Recommendation for your first simulation:** Use OSR x4 (20ms). This is reasonable for the terminal phase — you'll use IMU fusion for the high-noise shortfall. Make a note to test this setting's noise level empirically with your actual hardware before flying.

### 4.2 Sensor Read + State Machine Compute Cycle

Your Teensy runs a main loop that reads sensors, updates state, and makes decisions. The loop has a period — and your detection can only happen at the start of a new loop cycle.

Worst case: the ignition altitude is crossed at the very start of a loop cycle, so you don't detect it until the *next* cycle — a full loop period of delay.

**Design decision:** What loop rate will you run? Options:
- 100Hz → 10ms loop period
- 200Hz → 5ms loop period
- 500Hz → 2ms loop period (aggressive for Teensy + sensor fusion)

**For simulation:** Use 10ms (100Hz loop) as a conservative estimate. The Teensy 4.0 at 600MHz is more than capable of 200Hz+ with sensor fusion, but don't count on it until you measure it empirically.

**Research direction:** "Teensy 4.0 loop rate IMU" — the PJRC forums and many rocketry/drone projects have characterized this. For a simple state machine + IMU + barometer, 500Hz is achievable.

### 4.3 Relay or MOSFET Switching Latency

Between the Teensy sending a "fire" signal and current flowing through the e-match, there's a relay or transistor switching. This is typically very fast for a MOSFET (microseconds) but can be 10–20ms for a mechanical relay.

**Use a MOSFET** (logic-level N-channel MOSFET like the IRLZ44N), not a mechanical relay, for the ignition circuit. Switching time is then <1ms and you can ignore this term.

**Research direction:** Look up "model rocket ignition circuit MOSFET" — many amateur rocketry guides cover this. The key spec to look for is gate charge time at your driving voltage.

### 4.4 E-match Ignition Latency

This is the time between current flowing through the e-match and the match actually initiating (heating up enough to pop). This is not the same as motor spool time — it's the match itself.

**Typical range:** 10–50ms for common e-matches, measured at the firing current.
**Important:** This varies by e-match manufacturer and firing current. Higher current = faster ignition. The project brief explicitly says to measure this empirically on your bench before flying.

**For simulation:** Use 50ms as a conservative worst-case. Once you've characterized your specific e-matches with your specific firing circuit, replace this with measured values.

**Research direction:** "Estes e-match ignition delay" and "e-match firing time characterization" — some hobbyist rocketry groups have measured these. The key is to measure with your specific circuit configuration, not assume from datasheets.

### 4.5 Motor Spool Time (Thrust Onset Latency)

After the e-match fires, the motor propellant needs time to ignite and build up thrust. This is the time from chemical ignition to meaningful thrust (typically defined as when thrust exceeds vehicle weight — i.e., when the rocket actually starts decelerating).

**For Estes motors:** The thrust curve shows a sharp initial spike. However, meaningful deceleration (thrust > weight) may take 20–50ms from chemical ignition depending on the motor variant.

**Research direction:** Download the thrust curve data for your specific motor from [thrustcurve.org](https://www.thrustcurve.org). This site has digitized curves for virtually all certified hobby motors. Look at how quickly the thrust rises from zero — that rise time is your spool latency.

**For simulation:** Use 50ms as a conservative estimate. This is the number most likely to vary by motor variant.

### 4.6 Total Latency Stack

```
Barometer update period:          20ms  (OSR x4)
Loop cycle worst-case:            10ms  (100Hz loop)
MOSFET switching:                  1ms
E-match ignition:                 50ms
Motor spool to meaningful thrust: 50ms
─────────────────────────────────────
Worst-case total latency:        131ms
```

Your gross window (time from detection altitude to ground impact) must exceed this with comfortable margin. If gross window = 250ms and latency stack = 131ms, residual = 119ms. That's tight but workable — the brief says 100ms minimum before you need architecture changes.

If you can reduce loop latency (500Hz loop → 2ms) and characterize e-matches empirically (perhaps 30ms measured vs. 50ms assumed), your residual improves significantly.

---

## Part 5: The Script

Here is the complete simulation with annotations explaining every decision. Read each comment — the comments are the learning.

```python
import numpy as np
import matplotlib.pyplot as plt

# ─────────────────────────────────────────────
# VEHICLE PARAMETERS
# ─────────────────────────────────────────────

m = 0.50          # kg — rocket mass at landing (with landing motor, no ascent propellant)
                  # RESEARCH: weigh your components. Estes motor casing weights are in spec sheets.

Cd = 0.6          # drag coefficient during descent
                  # RESEARCH: OpenRocket will give you a number; use 0.6 as starting point.
                  # Run the simulation at 0.4 and 0.8 too — bracket the uncertainty.

diameter = 0.054  # m — outer diameter of body tube (54mm = standard Estes)
                  # RESEARCH: measure your actual tube OD when you have it.

A = np.pi * (diameter / 2)**2   # reference area in m² (cross-sectional circle)

h_apogee = 15.0  # m — apogee altitude above ground (15m ≈ 50ft, pessimistic starting estimate)
                 # RESEARCH: OpenRocket will give you the real number once your design is modeled.

h_detect = 3.0   # m — altitude at which barometer triggers ignition
                 # DESIGN DECISION: this is the number your control law targets.
                 # The simulation tells you whether this choice leaves enough window.

# ─────────────────────────────────────────────
# PHYSICAL CONSTANTS
# ─────────────────────────────────────────────

g = 9.81          # m/s² — gravitational acceleration
rho = 1.225       # kg/m³ — air density at sea level (fine for low-altitude flights)
                  # RESEARCH: if flying at significant elevation, look up density altitude correction.

# ─────────────────────────────────────────────
# LATENCY BUDGET (worst-case, in seconds)
# ─────────────────────────────────────────────

t_baro   = 0.020  # s — BMP388 update period at OSR x4
                  # RESEARCH: BMP388 datasheet Section 3.4 (output data rate table)

t_loop   = 0.010  # s — Teensy loop cycle worst-case (100Hz = 10ms)
                  # RESEARCH: measure your actual loop period with micros() in firmware

t_mosfet = 0.001  # s — MOSFET switching latency (negligible with logic-level FET)

t_ematch = 0.050  # s — e-match ignition latency (conservative estimate)
                  # RESEARCH: characterize empirically. Fire into a known load, measure current onset
                  # with an oscilloscope. Replace this number with your measured value.

t_spool  = 0.050  # s — motor spool time from chemical ignition to thrust > vehicle weight
                  # RESEARCH: download your motor's thrust curve from thrustcurve.org
                  # Find the time from t=0 to when thrust (N) crosses m*g (your vehicle weight)

total_latency = t_baro + t_loop + t_mosfet + t_ematch + t_spool

print(f"\n{'='*50}")
print(f"LATENCY BUDGET")
print(f"{'='*50}")
print(f"  Barometer update period:    {t_baro*1000:.1f} ms")
print(f"  Loop cycle (worst-case):    {t_loop*1000:.1f} ms")
print(f"  MOSFET switching:           {t_mosfet*1000:.1f} ms")
print(f"  E-match ignition:           {t_ematch*1000:.1f} ms")
print(f"  Motor spool to thrust:      {t_spool*1000:.1f} ms")
print(f"  ─────────────────────────────")
print(f"  TOTAL:                      {total_latency*1000:.1f} ms")

# ─────────────────────────────────────────────
# DESCENT SIMULATION (numerical integration)
# ─────────────────────────────────────────────

dt = 0.001        # s — time step (1ms). Small enough for good accuracy, fast to compute.

# Initial conditions
t = 0.0
v = 0.0           # m/s — velocity at apogee is 0 (momentarily stationary)
h = h_apogee      # m — start at apogee

# Storage for plotting
times = [t]
velocities = [v]
altitudes = [h]

# Track the detection event
detection_time = None
detection_velocity = None
detection_altitude = None
ground_impact_time = None

while h > 0:
    # Drag force (opposes motion — acts upward since rocket falls downward)
    F_drag = 0.5 * rho * v**2 * Cd * A   # Newtons

    # Net downward acceleration
    a = g - F_drag / m                    # m/s²

    # Euler integration: update velocity and position
    v = v + a * dt                        # velocity increases as rocket accelerates downward
    h = h - v * dt                        # altitude decreases

    t = t + dt

    times.append(t)
    velocities.append(v)
    altitudes.append(h)

    # Check if we've crossed the detection altitude
    if detection_time is None and h <= h_detect:
        detection_time = t
        detection_velocity = v
        detection_altitude = h

ground_impact_time = t

# ─────────────────────────────────────────────
# RESULTS
# ─────────────────────────────────────────────

gross_window = ground_impact_time - detection_time   # seconds from detection to ground
residual_window = gross_window - total_latency        # what's left after latency stack

print(f"\n{'='*50}")
print(f"VEHICLE PARAMETERS")
print(f"{'='*50}")
print(f"  Mass (landing config):      {m*1000:.0f} g")
print(f"  Drag coefficient:           {Cd:.2f}")
print(f"  Body diameter:              {diameter*1000:.0f} mm")
print(f"  Reference area:             {A*10000:.2f} cm²")

print(f"\n{'='*50}")
print(f"DESCENT PROFILE")
print(f"{'='*50}")
print(f"  Apogee:                     {h_apogee:.1f} m  ({h_apogee*3.281:.0f} ft)")
print(f"  Detection altitude:         {h_detect:.1f} m  ({h_detect*3.281:.0f} ft)")
print(f"  Detection velocity:         {detection_velocity:.2f} m/s  ({detection_velocity*3.281:.1f} ft/s)")
print(f"  Time from apogee to detect: {detection_time*1000:.0f} ms")
print(f"  Time from apogee to impact: {ground_impact_time*1000:.0f} ms")

print(f"\n{'='*50}")
print(f"IGNITION WINDOW ANALYSIS")
print(f"{'='*50}")
print(f"  Gross window (detect → impact):   {gross_window*1000:.1f} ms")
print(f"  Total latency stack:              {total_latency*1000:.1f} ms")
print(f"  RESIDUAL WINDOW:                  {residual_window*1000:.1f} ms")

if residual_window < 0:
    print(f"\n  ❌ FAIL — window is NEGATIVE. Architecture must change.")
    print(f"     Options: raise detection altitude, increase apogee, reduce latency stack,")
    print(f"     or add a drogue to slow descent.")
elif residual_window < 0.100:
    print(f"\n  ⚠️  MARGINAL — window is under 100ms. Proceed with caution.")
    print(f"     Abort logic is mandatory. Reduce latency where possible.")
elif residual_window < 0.200:
    print(f"\n  ✅ WORKABLE — 100–200ms residual. Measure all latency terms empirically.")
    print(f"     Apply early-ignition bias in control law.")
else:
    print(f"\n  ✅ GOOD — over 200ms residual. Design has margin.")

# ─────────────────────────────────────────────
# PLOT
# ─────────────────────────────────────────────

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle('Terminal Phase Descent Simulation', fontsize=13, fontweight='bold')

times_arr = np.array(times)
velocities_arr = np.array(velocities)
altitudes_arr = np.array(altitudes)

# Plot 1: Velocity vs Time
ax1.plot(times_arr * 1000, velocities_arr, 'b-', linewidth=2)
if detection_time:
    ax1.axvline(detection_time * 1000, color='orange', linestyle='--', label=f'Detection ({h_detect}m)')
    ax1.axvline((detection_time + total_latency) * 1000, color='red', linestyle='--',
                label=f'Thrust onset (after {total_latency*1000:.0f}ms latency)')
ax1.axvline(ground_impact_time * 1000, color='black', linestyle=':', label='Ground impact')
ax1.set_xlabel('Time from apogee (ms)')
ax1.set_ylabel('Descent velocity (m/s)')
ax1.set_title('Velocity vs Time')
ax1.legend(fontsize=8)
ax1.grid(True, alpha=0.3)

# Plot 2: Altitude vs Velocity
ax2.plot(velocities_arr, altitudes_arr, 'g-', linewidth=2)
if detection_velocity:
    ax2.axhline(h_detect, color='orange', linestyle='--', label=f'Detection altitude ({h_detect}m)')
    ax2.axhline(0, color='black', linestyle=':', label='Ground')
    ax2.plot(detection_velocity, h_detect, 'ro', markersize=8,
             label=f'Detection point: {detection_velocity:.1f} m/s')
ax2.set_xlabel('Descent velocity (m/s)')
ax2.set_ylabel('Altitude (m)')
ax2.set_title('Altitude vs Velocity (descent trajectory)')
ax2.legend(fontsize=8)
ax2.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('terminal_phase_sim.png', dpi=150, bbox_inches='tight')
plt.show()
print(f"\nPlot saved as terminal_phase_sim.png")
```

---

## Part 6: Running Sensitivity Analysis

One sim run at nominal values is not enough. You need to understand how the residual window changes with your key uncertainties. Add this loop after the main simulation:

```python
# ─────────────────────────────────────────────
# SENSITIVITY ANALYSIS
# ─────────────────────────────────────────────

print(f"\n{'='*60}")
print(f"SENSITIVITY TABLE (residual window in ms)")
print(f"{'='*60}")
print(f"{'':20} {'Cd=0.4':>10} {'Cd=0.6':>10} {'Cd=0.8':>10}")
print(f"{'─'*60}")

for h_ap in [10.0, 15.0, 20.0]:   # apogee altitudes to test (meters)
    results = []
    for cd_test in [0.4, 0.6, 0.8]:
        # Re-run sim with this apogee and Cd
        v2, h2 = 0.0, h_ap
        det_t2, imp_t2 = None, 0.0
        t2 = 0.0
        while h2 > 0:
            fd = 0.5 * rho * v2**2 * cd_test * A
            a2 = g - fd / m
            v2 += a2 * dt
            h2 -= v2 * dt
            t2 += dt
            if det_t2 is None and h2 <= h_detect:
                det_t2 = t2
            imp_t2 = t2
        res = (imp_t2 - det_t2 - total_latency) * 1000 if det_t2 else float('nan')
        results.append(f"{res:>10.0f}")
    print(f"  Apogee {h_ap:.0f}m ({h_ap*3.281:.0f}ft)   {''.join(results)}")

print(f"\nNegative = FAIL. 0-100ms = marginal. 100-200ms = workable. >200ms = good.")
```

This table tells you how much your design depends on getting Cd right, and whether a lower-than-expected apogee breaks you.

---

## Part 7: Design Decisions Summary — What You Need to Decide Before Running

Here are the explicit choices you need to make. Some require research, some are estimates to start.

| Parameter | Starting value | How to refine | Impact if wrong |
|-----------|---------------|---------------|-----------------|
| Mass at landing | Estimate from specs | Weigh actual parts | High — directly affects fall rate |
| Drag coefficient (Cd) | 0.6 (run 0.4–0.8) | OpenRocket model | Medium — affects terminal velocity |
| Apogee altitude | 15m (50ft) | OpenRocket simulation | High — more apogee = more window |
| Detection altitude | 3m (10ft) | Adjust based on results | High — core design variable |
| Barometer OSR setting | x4 (20ms) | Empirical noise test with hardware | Medium — latency vs. noise tradeoff |
| Teensy loop rate | 100Hz (10ms) | Measure with firmware | Medium |
| E-match latency | 50ms (conservative) | Bench measurement with oscilloscope | High — single biggest latency term |
| Motor spool time | 50ms | thrustcurve.org + thrust/weight calculation | High |

---

## Part 8: What to Do with the Results

**If residual > 200ms:** You have margin. Proceed to OpenRocket modeling. Lock your detection altitude and document it as a design parameter.

**If residual is 100–200ms:** Workable but sensitive. Two things to do before flying:
1. Characterize your e-match latency empirically (biggest single variable) — you may buy back 20–30ms.
2. Implement abort logic in the state machine: if IMU shows attitude error exceeding your threshold at detection altitude, suppress ignition.

**If residual < 100ms:** Architecture change required before any other work. Options in order of effectiveness:
- Increase detection altitude (fire earlier, from higher up)
- Increase apogee target (requires motor change or weight reduction)
- Add a small drogue chute to slow descent (adds complexity but dramatically increases window)
- Reduce sensor latency (switch to a faster IMU-primary approach for terminal phase)

**If residual is negative:** The project cannot work with this sensor stack at this altitude. This is not a failure — it's exactly what the simulation is for. You found it in week 1 rather than week 8.

---

## Part 9: Research Checklist Before Running

Before you run the simulation, look these up and fill in actual numbers:

- [ ] **BMP388 datasheet** — confirm output data rate table for your chosen OSR setting
- [ ] **Your motor's thrust curve** — download from thrustcurve.org (search your specific Estes motor part number). Find time-to-thrust-exceeds-weight.
- [ ] **Estimated rocket mass** — list every component and its weight in grams. Estes publishes motor casing weights.
- [ ] **Body tube outer diameter** — from your airframe supplier spec sheet
- [ ] **Estimated apogee** — use a simple impulse/mass estimate or wait for OpenRocket. F15-0 in a 500g rocket: roughly 15–25m.

---

*This guide is specific to this vehicle's constraints. The script is the deliverable — the guide explains every decision behind it.*
