# BIO2 · Active Ankle-Foot Orthosis Rehabilitation Bench

An interactive, browser-based teaching simulator for the **active ankle-foot orthosis (AAFO)** rehabilitation rig — a 1-DOF sagittal-plane ankle device driven by a DC servo motor, with a force-sensing resistor (FSR) under the foot and a rotary encoder on the ankle shaft.

The simulator models the orthosis as a **true two-body system** — an orthosis platform (motor side) and a foot (patient side) coupled through a stiff strap — and runs three rehabilitation control strategies from the thesis literature: repetitive control, admittance-resist, and admittance-assist.

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [The two-body physical model](#the-two-body-physical-model)
- [The three rehabilitation exercises](#the-three-rehabilitation-exercises)
- [Live demo](#live-demo)
- [Running locally](#running-locally)
- [Controls](#controls)
- [Architecture](#architecture)
- [Tuning the strap stiffness](#tuning-the-strap-stiffness)
- [Numerical notes](#numerical-notes)
- [File layout](#file-layout)
- [Reference](#reference)
- [License](#license)

---

## Why this exists

Rehabilitation engineering courses teach admittance control and repetitive control on paper, but students rarely get to *feel* the effect of a virtual mass, a learning gain, or a strap stiffness on the closed-loop behaviour. This simulator lets them:

- Watch the gait-cycle reference track sharpen cycle-by-cycle as repetitive control learns.
- Push the foot with a button (or a slider) and watch the FSR read the genuine strap load while the admittance law resists or assists.
- Drop the strap stiffness and see the foot slide inside the orthosis — the failure mode that a real AFO cuff prevents.
- Turn the motor off and feel the platform backdrive through the strap.

Everything runs client-side in a single HTML file — no server, no build step, no dependencies.

---

## The two-body physical model

### State (4-D)

| Symbol | Meaning | Side |
|---|---|---|
| `θ_P`, `ω_P` | orthosis platform angle / angular velocity | motor |
| `θ_F`, `ω_F` | foot angle / angular velocity | patient |

### Equations of motion

```
Platform :  JP·ω̇_P = T_motor − DP·ω_P − T_strap
Foot     :  JF·ω̇_F = T_strap  − DF·ω_F − G + T_patient
Strap    :  T_strap = kc·(θ_P − θ_F) + bd·(ω_P − ω_F)     [Nm]
FSR      :  F_fsr   = T_strap / FSR_ARM                    [N]
Gravity  :  G       = MF·g·RcF·cos(θ_F)                    (Eq. 4.25)
```

where:

- `J = M·Rc²` — inertia about the ankle shaft (foot and platform each have their own)
- `kc` — strap stiffness (torsion spring), default **600 Nm/rad**
- `bd` — strap internal viscous damping, kills high-frequency ringing
- `FSR_ARM` — lever converting strap torque to a linear FSR force

### Angle convention

- `0` = neutral foot flat
- `+` = dorsiflexion (toes up)
- `−` = plantarflexion (toes down)

ROM end-stops (anatomical, applied to both bodies): `+30°` dorsiflexion, `−45°` plantarflexion (thesis Fig. 4.19). Thesis assist/resist limits: `+29°` / `−30°`.

---

## The three rehabilitation exercises

| Exercise | Mode | Controller | Thesis defaults |
|---|---|---|---|
| **I · Passive** | repetitive control | learns a per-cycle feed-forward correction | `L = 0.9`, `Q = 0.97` |
| **II · Isometric** | admittance **RESIST** | heavy virtual mass — platform barely yields | `M = 1.0, B = 0.8, K = 1.0` |
| **III · Isotonic** | admittance **ASSIST** | light virtual mass — platform moves with patient | `M = 0.2, B = 0.8, K = 1.0` |

### Repetitive control (Exercise I)

Exploits that a gait cycle repeats: the tracking error from one period is used to correct the feed-forward command of the next, so tracking sharpens cycle after cycle.

```
u(i+N) = Q·( u(i) + L·e(i) )      e(i) = r(i) − θ_P(i)
```

Watch the **error trace** (lime): it is large on the first cycle, then collapses toward zero as the learned correction fills in. The thesis reports a full-cycle mean error of about 1.9° on hardware with an error-norm near 0.095.

### Admittance control (Exercises II & III)

Reads the strap load from the FSR and turns it into a platform-position command through a virtual mass-spring-damper (Newton's second law, thesis Eq. 4.27):

```
F(t) − K·x(t) − B·ẋ(t) = M·ẍ(t)
```

- **Resist** (heavy `M`): the platform barely yields, so the strap stretches as the foot tries to move, and the FSR reports that growing load — an isometric hold.
- **Assist** (light `M`): the platform moves freely with the strap load, so the muscle works through a range of motion against a near-constant load.

The FSR reading is **gravity-tared** so the admittance only responds to the patient's force, not the foot's own weight on the strap.

---

## Live demo

Open `afo_index.html` in any modern browser. No server, no build step.

Recommended first walk-through:

1. **Passive mode** (default) — press **Run**. Watch the platform (amber dashed arc) and foot (teal solid arc) track the gait-cycle reference together. The **STRAP SLIP** chip in the corner should stay green (sub-degree).
2. **Isometric mode** — press the **Hold to push foot** button (or use the Effort slider). Watch the FSR force jump while the foot barely moves. The strap slip stays under 1°.
3. **Drop the strap stiffness** `kc` in the controller parameters — watch the slip chip turn amber, then red, as the foot starts sliding inside the orthosis. Raise it back to confirm the coupling tightens.
4. **Turn the motor off** — the platform backdrives through the strap, and the foot swings freely under gravity and patient torque.

---

## Controls

### Patient interaction

| Control | What it does |
|---|---|
| **Hold to push foot** button (or Spacebar) | applies a full plantarflex / dorsiflex push to the foot |
| **Voluntary effort** slider | scales the patient torque from 0 to 100% of `PATIENT_MAX` |
| **Effort direction** slider | dorsiflex (up) or plantarflex (down) |
| **Orthosis motor ON** toggle | enables / disables the motor (motor off = backdrivable platform) |

### Gait cadence

| Control | What it does |
|---|---|
| **Cycle period** slider | gait-cycle period (default 2.0 s, range 1.2–3.5 s). The repetitive-control memory buffer `N = period / control period` rebuilds when this changes. |

### Controller parameters (per exercise)

| Exercise | Tunable |
|---|---|
| Passive | learning gain `L`, robustness filter `Q`, control period, strap stiffness `kc` |
| Isometric | virtual mass `M`, damping `B`, stiffness `K`, desired force, `kc` |
| Isotonic | virtual mass `M`, damping `B`, stiffness `K`, desired force, `kc` |
| Gait reference | sample period, `kc` |

---

## Architecture

```
afo_index.html  (single file, ~1300 lines)

├── <style>                     CSS — dark instrument-panel theme
├── <body>
│   ├── header + task rail      mode selector (passive / isometric / isotonic / gait)
│   ├── grid
│   │   ├── LEFT  card          bench view canvas + telemetry chips + controls
│   │   └── RIGHT card          scope canvas (4 instrument traces, 10 s window)
│   └── lower grid              prose explanation + syntax-highlighted Arduino sketch
└── <script>
    ├── P                       physical constants (foot + platform + strap + motor)
    ├── S                       simulation state (4-D two-body + sensors)
    ├── C                       controller parameters (per exercise)
    ├── gaitRef()               smoothstep-interpolated ankle gait-cycle profile
    ├── deriv() / rk4()         two-body dynamics + RK4 integrator
    ├── control()               outer loop: reference generation (RC / admittance)
    ├── innerPD()               inner PD loop: motor torque at the physics rate
    ├── admittanceStep()        virtual mass-spring-damper integration
    ├── step()                  outer physics step with sub-stepping
    ├── Scope class             4-track rolling-buffer oscilloscope
    ├── drawStage()             two-body canvas renderer + strap visualization
    ├── drawStrap()             visible rose strap band between the bodies
    └── UI wiring               tasks, params, run/reset, effort, push, telemetry
```

### Cascade control architecture

The strap raises the platform's effective natural frequency past what a 20 ms control loop can stabilize. The simulator therefore runs a **cascade**:

- **Outer loop** (20 ms, `ctrlDelay`): reference generation — repetitive-control correction or admittance output.
- **Inner loop** (0.25 ms, `dtPhys`): PD law that turns `(ref, θ_P)` into motor torque — like a real servo drive's inner torque loop.

This keeps the closed loop well-damped while the outer reference generator still ticks at the Arduino's actual loop rate.

---

## Tuning the strap stiffness

The **STRAP SLIP** chip in the bench-view corner is color-coded:

| Color | Slip range | Meaning |
|---|---|---|
| 🟢 green | `< 1°` | genuinely constrained (rigid cuff) |
| 🟡 amber | `1° – 5°` | soft strap, beginning to slide |
| 🔴 red | `> 5°` | sliding — the foot is escaping the orthosis |

### Default

`kc = 600 Nm/rad` — under a full 7.5 Nm patient push, the angular slip stays under 1°. The foot is genuinely constrained.

### Sweep (verified)

| `kc` | slip under full push | status |
|---|---|---|
| 100 | 5.07° | sliding |
| 300 | 1.70° | sliding |
| **600** | **0.85°** | **constrained (default)** |
| 1200 | 0.42° | very tight |

Slip scales as `T_patient / kc`, confirming the spring model is working correctly.

---

## Numerical notes

- **Integrator**: 4th-order Runge-Kutta (RK4).
- **Outer physics step**: `dt = 5 ms` (matches the Arduino control period).
- **Inner sub-step**: `dtPhys = 0.25 ms` — the strap is very stiff (`ω = sqrt(kc·(1/JF + 1/JP)) ≈ 800 rad/s` at `kc = 600`), so we sub-step to keep `ω·dt < 0.2` for RK4 stability.
- **End-stops**: anatomical ROM limits applied to **both** bodies (inelastic bounce-to-stop).
- **Stability**: verified over 60 s with sinusoidal reference + sinusoidal patient push — no NaN, no energy blow-up.

---

## File layout

```
.
├── README.md                   this file
├── download/
│   └── afo_index.html          the simulator (single file, open in a browser)
└── scripts/
    ├── sanity_check.js         headless physics verification (8 tests)
    └── extract.js              helper: extracts <script> for syntax check
```

### Running the sanity checks

```bash
node scripts/sanity_check.js
```

Runs 8 tests covering: gravity load at neutral, full patient push with motor on, motor-off backdriving, dorsiflex push symmetry, 60 s numerical stability, strap-stiffness sweep, admittance gravity-tare, and admittance-with-push. All should pass.

---

## Reference

Mansour, M. (2023). *Development and Control of Lower Limb Active Orthosis for Athlete Rehabilitation* [PhD thesis, Sakarya University of Applied Sciences]. Key equations referenced in the simulator:

- Eq. 4.22–4.26 — Euler-Lagrange ankle dynamics
- Eq. 4.27–4.33 — admittance virtual mass-spring-damper
- Eq. 4.39 — repetitive-control update law `u(i+N) = Q·(u(i) + L·e(i))`
- Fig. 4.19 — ROM end-stops
- Fig. 4.29 / 5.3 — ankle gait-cycle reference profile
- Fig. 5.1 — hardware (Arduino Uno, 24 V DC servo, 360° encoder, FSR)

Hardware mirrored in the Arduino sketch listings: Arduino Uno (PWM pin 9), 24 V DC servo motor, 360° rotary encoder, FSR under the foot.

---

## License

Released for educational use. If you build on this in a published work, please cite the underlying thesis:

> Mansour, M. (2023). *Development and Control of Lower Limb Active Orthosis for Athlete Rehabilitation*. PhD thesis, Sakarya University of Applied Sciences.
