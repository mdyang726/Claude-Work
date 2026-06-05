# Terminal Phase Simulation — Step-by-Step Guide

*How to write, run, and interpret the ignition window feasibility script*

---

## What This Script Does

One question: **at the moment your IMU integration detects the ignition altitude, how much time remains before ground impact after subtracting every real-world delay between detection and meaningful thrust?**

If that number is positive with margin → proceed. If it's near zero or negative → architecture must change before you order anything.

The simulation models:
1. Your rocket falling from apogee under gravity + aerodynamic drag
2. Velocity and altitude at every millisecond of descent
3. The moment your IMU-integrated altitude crosses the ignition threshold
4. Subtraction of every real latency in your system
5. What's left — the residual window

---

## How to Run This in VS Code

**One-time setup — paste this into the VS Code terminal (`Ctrl+\``):**

```
pip install numpy matplotlib
```

Then create a new file called `terminal_sim.py`, paste in the script from Part 5, and press the Play button (▶) in the top right, or run:

```
python terminal_sim.py
```

That's it. The script prints results to the terminal and saves a plot as `terminal_phase_sim.png` in the same folder.

---

## Part 1: The Physics Model

### 1.1 Forces During Descent

After apogee (launch motor ejected, landing motor not yet ignited), two forces act on the rocket:

- **Gravity** pulls it down: `F_gravity = m × g`
- **Drag** resists motion (acts upward): `F_drag = 0.5 × ρ × v² × Cd × A`

Net downward acceleration at any moment:

```
a = g − (ρ × v² × Cd × A) / (2 × m)
```

### 1.2 Altitude via IMU Double Integration

The flight computer does not use a barometer. Instead, it integrates IMU vertical acceleration twice each loop cycle:

```
Step 1:  v_new = v + a_measured × dt    → velocity
Step 2:  h_new = h − v_new × dt        → altitude (decreasing as rocket falls)
```

The integrator is reset to zero at apogee (IMU detects velocity = 0). From that point, only the short descent arc needs to be integrated, keeping drift negligible.

The simulation uses the same integration approach — it models the descent the same way the flight computer will see it.

### 1.3 Numerical Integration (Simulation)

You can't solve the drag equation analytically because drag depends on velocity. Instead, step forward at dt = 1ms:

```
At each timestep:
  compute drag from current velocity
  compute net acceleration
  update velocity and altitude
  if altitude ≤ h_detect: record detection moment
  if altitude ≤ 0: stop (ground impact)
```

~2000 steps for a ~2-second descent. Runs instantly.

---

## Part 2: Vehicle Parameters

### 2.1 Mass at Landing (`m`)

Rocket mass **when the landing motor fires** — after the launch motor has been ejected.

At landing you have: airframe + fins + electronics + legs + landing motor (full, unfired). No launch motor or its propellant.

**Estimate:** 400–600g to start. Weigh everything when you have parts. Estes publishes casing and propellant masses in spec sheets.

**Why it matters:** Higher mass = faster terminal velocity = less time to impact.

### 2.2 Drag Coefficient (`Cd`)

A dimensionless number for how much drag your shape produces. Typical for an unpowered model rocket in descent: **0.4–0.8**.

Descent Cd is higher than ascent Cd because:
- Attitude during passive descent is uncertain (low CoM produces pendulum stability, not locked nose-first)
- Legs may be partially deployed
- Launch motor ejected, changing the base geometry

**Run the simulation at Cd = 0.4, 0.6, and 0.8** to bracket the uncertainty. OpenRocket will give you a geometry-based estimate later — use that to narrow the range.

### 2.3 Reference Area (`A`)

Cross-sectional area of the body tube:

```
A = π × r²     where r = outer radius in meters
```

Example: 54mm tube → r = 0.027m → A ≈ 0.00229 m²

Standard Estes F-class motors require a 29mm mount, which fits inside a 54mm body tube.

### 2.4 Apogee Altitude (`h_apogee`)

Where the simulation starts. Velocity = 0 at apogee.

**Key insight:** Higher apogee directly expands your ignition window — more altitude to fall means slower descent velocity at the ignition point and more time before ground impact. Apogee is a design variable, not fixed. 30ft was a reference video constraint; you are not bound to it.

**Starting value:** 15m (50ft) — pessimistic, on the low end. If the design passes here, it passes everywhere above. Once you have an OpenRocket model, replace with the simulated apogee.

### 2.5 Detection Altitude (`h_detect`)

The IMU-integrated altitude at which the state machine triggers ignition. This is a **design variable you choose** — your control law maps measured apogee to this threshold.

**Starting point:** 3m (10ft). The sensitivity analysis will tell you how this choice interacts with apogee and Cd.

**The tradeoff:** Higher detection altitude = more time between detection and impact (larger gross window) but lower descent velocity at ignition, which means the motor may not fully arrest descent before touchdown. Lower detection altitude = less window but the motor fires into faster descent. The simulation reveals this tradeoff quantitatively.

---

## Part 3: The Latency Budget

Every millisecond between "detection" and "thrust exceeds vehicle weight" must be subtracted from your gross window. Stack these in order.

### IMU Integration Cycle

The IMU produces new acceleration data at its configured output rate. At 200Hz, a new reading arrives every 5ms. The flight computer integrates each reading as it arrives.

Worst-case: ignition altitude is crossed at the very start of a loop cycle, so detection happens at the *next* cycle. Latency = one loop period.

**At 200Hz loop: 5ms. At 100Hz loop: 10ms.**

Unlike a barometer (which required a separate wait period on top of the loop cycle), the IMU integration IS the loop cycle. This is why the latency budget improves significantly over a barometer-based system.

**Measure your actual Teensy loop rate** with `micros()` in firmware once your sensor fusion code is written. The number you use in this sim should match measured reality.

### MOSFET Switching

Logic-level N-channel MOSFET (e.g., IRLZ44N): switching time < 1ms. Use a MOSFET, not a mechanical relay. Relay switching adds 10–20ms unnecessarily.

### E-match Ignition Latency

Time from current-through-match to chemical initiation. **Typical: 10–50ms.** Varies significantly by manufacturer and firing current.

**This is the single largest variable latency and must be measured empirically** — fire your specific e-matches with your specific circuit into a resistive load, measure with an oscilloscope or a microcontroller timer pin. Do this before flying. Use 50ms in the simulation as a conservative placeholder until measured.

### Motor Spool Time

Time from chemical ignition to thrust exceeding vehicle weight (the point where the rocket actually starts decelerating). For Estes F-class motors, the thrust curve shows a sharp initial rise — but "meaningful thrust" depends on your vehicle weight.

**How to find it:** Download your motor's thrust curve from thrustcurve.org. Find the time at which thrust (N) first exceeds `m × g` (your vehicle weight in Newtons). That time is your spool latency.

**For simulation:** Use 50ms conservative until you've done the thrust curve calculation.

### Total Stack

```
IMU loop cycle (200Hz):           5ms
MOSFET switching:                  1ms
E-match ignition (conservative):  50ms
Motor spool to thrust > weight:   50ms
──────────────────────────────────────
Total worst-case:                106ms
```

At 100Hz loop:

```
IMU loop cycle (100Hz):           10ms
MOSFET switching:                  1ms
E-match ignition:                 50ms
Motor spool:                      50ms
──────────────────────────────────────
Total:                           111ms
```

Either way, the budget is tighter than a barometer-based system (which added 20ms on top). **Your gross window must exceed this with comfortable margin.**

---

## Part 4: File and Folder Setup

Before pasting the script, set up a clean folder:

```
model-rocket/
├── terminal_sim.py        ← paste the script here
├── terminal_phase_sim.png ← generated automatically when you run it
```

Open the folder in VS Code: `File → Open Folder`, select it. Then create `terminal_sim.py` and paste in the script below.

---

## Part 5: The Script

Paste this entire block into `terminal_sim.py`. All parameters you need to change are in the top section labeled **VEHICLE PARAMETERS** and **LATENCY BUDGET** — everything below those sections runs automatically.

```python
import numpy as np
import matplotlib.pyplot as plt

# ═══════════════════════════════════════════════════════════════
# VEHICLE PARAMETERS  ← change these to match your rocket
# ═══════════════════════════════════════════════════════════════

m = 0.35          # kg — rough estimate for a 3D-printed TVC rocket at 54mm diameter
                  # Component breakdown (estimates):
                  #   Airframe + nose cone + fins (PLA/PETG):  ~80g
                  #   Teensy 4.0 + IMU + wiring:               ~30g
                  #   Micro servo(s) for TVC gimbal:           ~20g
                  #   LiPo battery (1S–2S):                    ~40g
                  #   Landing motor (Estes F, unfired):        ~60g
                  #   4× spring-loaded legs (stowed on ascent):~30g
                  #   Misc hardware, connectors, pyro:         ~25g
                  #                              Rough total: ~285g
                  # Use 0.35 (350g) as a conservative/heavier starting estimate.
                  # NOTE: launch motor has already ejected at this point — do not
                  # include its mass here. Only the landing motor counts.
                  # HOW TO REFINE: list every component and weigh it. Mass is the
                  # parameter this sim is most sensitive to.

Cd = 0.6          # drag coefficient during passive descent (dimensionless)
                  # Legs are STOWED during ascent so ascent Cd is lower (~0.4).
                  # For descent, legs are deployed — this increases Cd significantly.
                  # Reasonable descent estimates:
                  #   0.5 — legs deployed, nose-down attitude, clean shape
                  #   0.7 — legs deployed, uncertain attitude (low-CoM pendulum)
                  #   0.9 — worst case: tumbling or high-drag attitude
                  # RUN THE SENSITIVITY ANALYSIS at 0.4 / 0.6 / 0.8.
                  # HOW TO REFINE: OpenRocket computes Cd from your geometry.

diameter = 0.054  # m — 54mm outer diameter (standard Estes BT-60 equivalent)
                  # WHY NOT 27mm: Estes F-class motors are 29mm OD. A 27mm tube
                  # is narrower than the motor itself. Once you add a motor mount
                  # and tube wall, minimum practical airframe is ~40mm for the
                  # motor alone. The TVC gimbal and electronics push this further.
                  # 54mm: tight but workable. Requires micro servos and compact
                  #        electronics sled. Least drag penalty.
                  # 75mm: easier to build and wire. More gimbal room. More drag.
                  # Recommend starting at 54mm; go to 75mm only if the gimbal
                  # geometry forces it during CAD.

A = np.pi * (diameter / 2)**2   # reference area (m²) — computed automatically

h_apogee = 15.0  # m — apogee altitude above ground (15m ≈ 50ft)
                 # This is a DESIGN VARIABLE, not a fixed constraint.
                 # Higher apogee = larger ignition window (slower descent at ignition,
                 # more time before ground impact). 30ft from the reference video was
                 # that builder's constraint — you are not bound to it.
                 # Plausible range for Estes F-class in a 350g rocket: 15–40m.
                 # START WITH 15m (pessimistic). If the window is tight here, raise it.
                 # HOW TO REFINE: OpenRocket once you have a CAD model.

h_detect = 3.0   # m — IMU-integrated altitude that triggers ignition
                 # THIS IS A DESIGN VARIABLE. Start at 3m (≈10ft).
                 # Sensitivity analysis (Part 6) shows how changing this trades off
                 # window size against descent velocity at ignition.


# ═══════════════════════════════════════════════════════════════
# LATENCY BUDGET  ← update these once you have measured values
# ═══════════════════════════════════════════════════════════════

t_loop   = 0.005  # s — flight computer loop period (5ms = 200Hz loop)
                  # This is also your IMU integration cycle — no separate barometer
                  # wait needed. The IMU integrates on every loop cycle.
                  # HOW TO MEASURE: time your firmware loop with micros() on Teensy.
                  # Use 0.010 (100Hz) if you haven't measured yet — conservative.

t_mosfet = 0.001  # s — MOSFET switching latency (<1ms with logic-level FET)
                  # Use a MOSFET (e.g. IRLZ44N), not a relay. Relay adds 10-20ms.

t_ematch = 0.050  # s — e-match ignition latency (50ms = conservative placeholder)
                  # THIS MUST BE MEASURED on your bench before flying.
                  # Fire your e-matches with your circuit into a resistor load.
                  # Measure time from trigger signal to current onset with a scope
                  # or Teensy timer pin. Replace 0.050 with your measured value.

t_spool  = 0.050  # s — motor spool time from ignition to thrust > vehicle weight
                  # HOW TO GET IT: download your motor's thrust curve from
                  # thrustcurve.org. Find the time where thrust (N) first exceeds
                  # m * 9.81 (your vehicle weight in Newtons).

total_latency = t_loop + t_mosfet + t_ematch + t_spool


# ═══════════════════════════════════════════════════════════════
# PHYSICAL CONSTANTS  ← leave these alone
# ═══════════════════════════════════════════════════════════════

g   = 9.81    # m/s²
rho = 1.225   # kg/m³ — sea level air density


# ═══════════════════════════════════════════════════════════════
# PRINT SETUP
# ═══════════════════════════════════════════════════════════════

print(f"\n{'='*52}")
print(f"LATENCY BUDGET")
print(f"{'='*52}")
print(f"  IMU loop cycle (detection):     {t_loop*1000:.1f} ms")
print(f"  MOSFET switching:               {t_mosfet*1000:.1f} ms")
print(f"  E-match ignition:               {t_ematch*1000:.1f} ms")
print(f"  Motor spool to thrust>weight:   {t_spool*1000:.1f} ms")
print(f"  {'─'*38}")
print(f"  TOTAL:                          {total_latency*1000:.1f} ms")


# ═══════════════════════════════════════════════════════════════
# DESCENT SIMULATION
# ═══════════════════════════════════════════════════════════════
# Models the rocket falling from apogee under gravity + drag.
# Same physics the flight computer integrates onboard.
# dt=1ms gives good accuracy and runs instantly.

dt = 0.001

t, v, h = 0.0, 0.0, h_apogee

times, velocities, altitudes = [t], [v], [h]
detection_time = detection_velocity = ground_impact_time = None

while h > 0:
    F_drag = 0.5 * rho * v**2 * Cd * A
    a      = g - F_drag / m
    v      = v + a * dt
    h      = h - v * dt
    t      = t + dt

    times.append(t)
    velocities.append(v)
    altitudes.append(h)

    if detection_time is None and h <= h_detect:
        detection_time     = t
        detection_velocity = v

ground_impact_time = t


# ═══════════════════════════════════════════════════════════════
# RESULTS
# ═══════════════════════════════════════════════════════════════

gross_window    = ground_impact_time - detection_time
residual_window = gross_window - total_latency

print(f"\n{'='*52}")
print(f"VEHICLE PARAMETERS")
print(f"{'='*52}")
print(f"  Mass (landing config):          {m*1000:.0f} g")
print(f"  Drag coefficient:               {Cd:.2f}")
print(f"  Body diameter:                  {diameter*1000:.0f} mm")
print(f"  Reference area:                 {A*10000:.2f} cm²")

print(f"\n{'='*52}")
print(f"DESCENT PROFILE")
print(f"{'='*52}")
print(f"  Apogee:                         {h_apogee:.1f} m  ({h_apogee*3.281:.0f} ft)")
print(f"  Detection altitude:             {h_detect:.1f} m  ({h_detect*3.281:.0f} ft)")
print(f"  Velocity at detection:          {detection_velocity:.2f} m/s  ({detection_velocity*3.281:.1f} ft/s)")
print(f"  Time apogee → detection:        {detection_time*1000:.0f} ms")
print(f"  Time apogee → ground:           {ground_impact_time*1000:.0f} ms")

print(f"\n{'='*52}")
print(f"IGNITION WINDOW")
print(f"{'='*52}")
print(f"  Gross window (detect→impact):   {gross_window*1000:.1f} ms")
print(f"  Total latency stack:            {total_latency*1000:.1f} ms")
print(f"  RESIDUAL WINDOW:                {residual_window*1000:.1f} ms")

if residual_window < 0:
    print(f"\n  FAIL — window is NEGATIVE. Architecture must change.")
    print(f"  Options: raise h_apogee, raise h_detect, reduce e-match latency.")
elif residual_window < 0.100:
    print(f"\n  MARGINAL — under 100ms. Measure all latency terms empirically.")
    print(f"  Raise apogee or detection altitude before flying.")
elif residual_window < 0.200:
    print(f"\n  WORKABLE — 100-200ms. Measure e-match latency on bench.")
    print(f"  Apply early-ignition bias in control law.")
else:
    print(f"\n  GOOD — over 200ms. Solid margin.")


# ═══════════════════════════════════════════════════════════════
# PLOT
# ═══════════════════════════════════════════════════════════════

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
fig.suptitle('Terminal Phase Descent Simulation', fontsize=13, fontweight='bold')

times_arr      = np.array(times)
velocities_arr = np.array(velocities)
altitudes_arr  = np.array(altitudes)

# Left: velocity vs time
ax1.plot(times_arr * 1000, velocities_arr, 'b-', linewidth=2)
ax1.axvline(detection_time * 1000, color='orange', linestyle='--',
            label=f'Detection ({h_detect}m)')
ax1.axvline((detection_time + total_latency) * 1000, color='red', linestyle='--',
            label=f'Thrust onset (+{total_latency*1000:.0f}ms latency)')
ax1.axvline(ground_impact_time * 1000, color='black', linestyle=':', label='Ground')
ax1.set_xlabel('Time from apogee (ms)')
ax1.set_ylabel('Descent velocity (m/s)')
ax1.set_title('Velocity vs Time')
ax1.legend(fontsize=8)
ax1.grid(True, alpha=0.3)

# Right: altitude vs velocity
ax2.plot(velocities_arr, altitudes_arr, 'g-', linewidth=2)
ax2.axhline(h_detect, color='orange', linestyle='--',
            label=f'Detection altitude ({h_detect}m)')
ax2.axhline(0, color='black', linestyle=':', label='Ground')
ax2.plot(detection_velocity, h_detect, 'ro', markersize=8,
         label=f'Detection: {detection_velocity:.1f} m/s')
ax2.set_xlabel('Descent velocity (m/s)')
ax2.set_ylabel('Altitude (m)')
ax2.set_title('Altitude vs Velocity')
ax2.legend(fontsize=8)
ax2.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('terminal_phase_sim.png', dpi=150, bbox_inches='tight')
plt.show()
print(f"\nPlot saved → terminal_phase_sim.png")
```

---

## Part 6: Sensitivity Analysis

Run this as a **separate script** or append it to the bottom of `terminal_sim.py`. It sweeps apogee altitude and Cd to show you the full design space.

Paste this into `sensitivity.py`:

```python
import numpy as np

# ═══════════════════════════════════════════════════════════════
# COPY THESE EXACTLY FROM terminal_sim.py
# ═══════════════════════════════════════════════════════════════

m        = 0.35
diameter = 0.054
A        = np.pi * (diameter / 2)**2
h_detect = 3.0
dt       = 0.001
g        = 9.81
rho      = 1.225

t_loop   = 0.005
t_mosfet = 0.001
t_ematch = 0.050
t_spool  = 0.050
total_latency = t_loop + t_mosfet + t_ematch + t_spool

# ═══════════════════════════════════════════════════════════════
# SWEEP PARAMETERS  ← edit these ranges as needed
# ═══════════════════════════════════════════════════════════════

apogee_values = [10.0, 15.0, 20.0, 30.0]   # meters
cd_values     = [0.4, 0.6, 0.8]            # drag coefficients

# ═══════════════════════════════════════════════════════════════
# SWEEP
# ═══════════════════════════════════════════════════════════════

def run_sim(h_ap, cd):
    v, h, t = 0.0, h_ap, 0.0
    det_t = imp_t = None
    while h > 0:
        fd  = 0.5 * rho * v**2 * cd * A
        a   = g - fd / m
        v  += a * dt
        h  -= v * dt
        t  += dt
        if det_t is None and h <= h_detect:
            det_t = t
        imp_t = t
    res_ms = (imp_t - det_t - total_latency) * 1000 if det_t else float('nan')
    return res_ms

print(f"\n{'='*62}")
print(f"SENSITIVITY TABLE — residual window (ms)")
print(f"Positive = margin remaining. Negative = FAIL.")
print(f"{'='*62}")

header = f"{'':22}" + "".join(f"  Cd={cd:.1f}" for cd in cd_values)
print(header)
print("─" * 62)

for h_ap in apogee_values:
    row = f"  Apogee {h_ap:.0f}m ({h_ap*3.281:.0f}ft)   "
    for cd in cd_values:
        res = run_sim(h_ap, cd)
        flag = " ✗" if res < 0 else (" ?" if res < 100 else "")
        row += f"  {res:>6.0f}ms{flag:<2}"
    print(row)

print(f"\n  < 0ms  = FAIL (architecture change required)")
print(f"  0–100ms = marginal (measure all latency terms, raise apogee)")
print(f"  100–200ms = workable (measure e-match, apply early bias)")
print(f"  > 200ms = good margin")
```

---

## Part 7: Parameters at a Glance

| Parameter | Starting value | How to refine | Sensitivity |
|-----------|---------------|---------------|-------------|
| Mass at landing | ~350g (see breakdown in script) | Weigh actual parts | High — affects fall rate directly |
| Drag coefficient | 0.6 descent (legs deployed); run 0.4–0.8 | OpenRocket geometry model | Medium |
| Body tube diameter | 54mm (27mm is too small — motor is 29mm OD) | Set by gimbal CAD | Low once chosen |
| Apogee altitude | 15m (50ft) | OpenRocket sim | High — biggest lever on window size |
| Detection altitude | 3m (10ft) | Adjust from sensitivity table | High — core design variable |
| IMU loop rate | 200Hz (5ms) | Measure with `micros()` in firmware | Medium |
| E-match latency | 50ms (conservative) | Bench measurement with scope | High — single biggest variable |
| Motor spool time | 50ms | thrustcurve.org thrust/weight calc | High |

---

## Part 8: What to Do With the Results

**Residual > 200ms:** Good margin. Proceed to OpenRocket and hardware ordering.

**Residual 100–200ms:** Workable but tight. Before flying:
1. Measure e-match latency empirically — you may recover 20–30ms.
2. Implement abort logic: if IMU shows attitude error above threshold at detection altitude, suppress ignition.
3. Apply early-ignition bias explicitly in the control law.

**Residual < 100ms:** Architecture change before anything else. In order of effectiveness:
- Raise `h_apogee` (more motor impulse, lighter airframe, or accept higher target altitude)
- Raise `h_detect` (fire earlier — but verify the motor can still arrest descent from that higher altitude)
- Run at 200Hz+ loop rate to reduce loop latency

**Residual negative:** The design cannot work at this altitude with this sensor stack. This is exactly what the simulation is for — found in week 1, not week 8.

---

## Part 9: Research Checklist Before Running

- [ ] **Your motor's thrust curve** — download from thrustcurve.org (search your Estes motor part number). Find time-to-thrust > vehicle weight. This is `t_spool`.
- [ ] **Estimated rocket mass** — list every component in grams. Estes publishes motor casing and propellant masses.
- [ ] **Body tube outer diameter** — from your airframe supplier or Estes spec sheet.
- [ ] **IMU datasheet** — confirm maximum output data rate at your chosen configuration. This sets your achievable loop rate.
- [ ] **Apogee estimate** — F15-0 in a 500g rocket: roughly 15–25m. Use 15m to start; replace with OpenRocket once you have a model.

---

*This guide is specific to this vehicle. The script is the deliverable — every parameter in it is a design decision or a measurement you need to make.*
