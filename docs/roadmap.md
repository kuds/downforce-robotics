# Autonomous Racing Car: Simulation → Real-World Roadmap

A staged plan for designing a track in a 3D environment, training a car to lap
it as fast as possible, accounting for aerodynamics, and ultimately building
the physical vehicle. Scoped so each phase produces a usable artifact even if
later phases slip.

---

## Phase 0 — Project framing (1–2 weeks)

Decide the constraints before touching code. They drive every later choice.

- **Vehicle scale.** 1/10 RC (Traxxas / F1TENTH class), 1/8 buggy, or full
  Roborace-style? Most learnable path is **1/10 — F1TENTH platform** because
  the mechanical, electrical, and software stacks are documented and parts are
  cheap (~$3.5–4k all-in for a full build).
- **Track type.** Indoor flat with cones, outdoor karting circuit, or a fixed
  marked track? Indoor flat is by far the easiest first target — it removes
  weather, lighting, and surface variance.
- **Sensors.** Pick early; aero and chassis design depend on what's mounted.
  Typical F1TENTH stack: 2D LiDAR (Hokuyo UST-10LX or RPLIDAR S2), stereo or
  RGB-D camera (Intel RealSense D435i), IMU, wheel encoders.
- **Compute.** NVIDIA Jetson Orin Nano (8GB) is the current default for
  on-vehicle inference. If you're running pure classical control + LiDAR, a
  Raspberry Pi 5 is enough.
- **Win condition.** Define lap time and reliability targets up front
  (e.g. "5 consecutive clean laps under 12s on a 40m track").

**Deliverable:** one-page spec with vehicle class, sensors, compute, target
metrics, and budget.

---

## Phase 1 — Pick the simulator (1 week)

The simulator choice constrains everything downstream — physics fidelity, RL
ergonomics, and how cleanly you can transfer to real hardware.

| Simulator | Strengths | Weaknesses | Best for |
|---|---|---|---|
| **F1TENTH Gym** (Python, 2D) | Lightweight, pure Gym API, fast | 2D only, no aero, no visuals | Early RL + planner prototyping |
| **CARLA** | High-fidelity urban driving, ROS bridge | Heavy, oriented to road cars | Camera-based perception |
| **AWS DeepRacer Console** | Zero setup, easy RL onboarding | Closed, fixed action space | Learning RL concepts |
| **NVIDIA Isaac Sim / Lab** | Best-in-class GPU sim, domain randomization, sim2real focus | Steep learning curve, heavy hardware | Serious sim2real RL |
| **Gazebo (Harmonic) + ROS 2** | Standard robotics stack | Mediocre physics for racing | Integration testing |
| **Unity ML-Agents** | Easy 3D scene authoring, good RL bindings | Less robotics-native | Custom tracks + visual training |
| **Project CHRONO** | Extremely accurate vehicle dynamics, terrain | Not RL-first, integration work | Tire / suspension validation |

**Recommendation:** start with **F1TENTH Gym** (planner + classical
controller baseline), then move to **Isaac Lab** for visual RL once the
problem is well-posed. Use **Project CHRONO** offline to validate tire model
parameters before committing to Isaac.

---

## Phase 2 — Build the 3D track (1–2 weeks)

You need both a **drivable mesh** (for the simulator's physics/collision) and
a **centerline + boundary description** (for lap timing, reward shaping, and
reference trajectories).

**Authoring options:**
- **Blender** — author the road surface, kerbs, walls. Export `.obj`/`.fbx`.
  Use the *Sapling* add-on or *Geometry Nodes* for environment dressing.
- **OpenStreetMap → road mesh** via `osmnx` if you want to mirror a real
  karting circuit.
- **Procedural** — generate a centerline (Catmull-Rom spline through random
  control points), extrude to a width, mesh it. Lets you train on infinite
  randomized tracks, which is critical for generalization.

**Track description format:** keep a CSV/YAML with `(x, y, width_left,
width_right, banking)` per centerline waypoint. This is the canonical
representation; everything else (mesh, occupancy grid, raceline) is derived.

**Tooling:**
- Blender 4.x for authoring
- `trimesh` (Python) for mesh post-processing
- `numpy-quadprog` or `casadi` for offline raceline optimization (minimum
  curvature → minimum lap time)

---

## Phase 3 — Vehicle dynamics model (2 weeks)

Garbage dynamics → garbage policy. Pick the right fidelity for the question
you're answering.

1. **Kinematic bicycle** — fine for pathfinding sanity checks, fails at the
   limit of grip.
2. **Dynamic bicycle + Pacejka tire model** — the F1TENTH default. Captures
   slip and is what you'll race on.
3. **Multi-body (Project CHRONO / Isaac PhysX)** — needed if suspension,
   weight transfer, or aero balance matter for your win condition.

**Parameters to identify:**
- Mass, CoG, wheelbase, track width
- Tire stiffness (longitudinal `Cα`, lateral `Cγ`) — measure on a roller rig
  or fit from telemetry of a hand-driven lap
- Motor/ESC torque curve — bench-test, log current and rpm
- Steering rack: max angle, slew rate, deadband

**Validate** by hand-driving the real car around a known path, then replaying
the same control input in sim and comparing trajectories. They should match
within ~10% over 5 seconds of open-loop driving before you trust RL results.

---

## Phase 4 — Aerodynamics and CFD (parallel, 2–4 weeks)

For 1/10 scale at ≤15 m/s, aero is **secondary** to tires and weight transfer
— but downforce-generating bodywork still meaningfully shifts cornering
limits, and you said "downforce" is in the brand name, so:

**Tooling:**
- **OpenFOAM** (free, open source, industry standard) — `simpleFoam` for
  steady-state RANS, `pimpleFoam` for transient. Run in Docker.
- **SU2** — alternative open-source CFD, friendlier Python bindings.
- **ParaView** for visualization.
- **SimScale** (browser-based, has free tier) if you want to skip the
  OpenFOAM learning curve.
- **Ansys Discovery / Fluent Student** — free for educational use, much nicer
  UX, capped mesh size.

**Workflow:**
1. CAD the body in **Fusion 360** (free for hobbyists/students) or
   **OnShape** (free for public docs).
2. Export STL → mesh in `snappyHexMesh` or `cfMesh`.
3. Steady RANS at three speeds (5, 10, 15 m/s), three yaw angles
   (0°, 5°, 10°). 9 runs, ~1–4 hours each on a workstation.
4. Extract `Cd`, `Cl_front`, `Cl_rear`, side force `Cy`. Tabulate vs. yaw.
5. **Plug those coefficients into the sim's vehicle model** as
   speed-dependent forces — that's how aero affects training. Don't try to
   couple full CFD into the RL loop; it's 10⁶× too slow.

**Physical validation:** smoke test in a small wind tunnel (community college
labs often have one) or instrument the car with a pitot tube + load cells on
the suspension to back out downforce on a long straight.

---

## Phase 5 — Controller + RL training (3–6 weeks)

**Always build a non-RL baseline first.** It sets the bar and serves as the
safety controller during real-world testing.

**Baseline stack:**
- Localization: particle filter on LiDAR vs. occupancy grid (`particle_filter`
  ROS 2 package) or SLAM Toolbox.
- Planner: precomputed minimum-curvature raceline + Pure Pursuit or Stanley
  controller. This is the F1TENTH "lab 1–4" stack and is genuinely fast.
- MPC: `acados` or `do-mpc` for a Model Predictive Contouring Controller —
  this is what wins F1TENTH races today.

**RL stack (once baseline works):**
- Framework: **Stable-Baselines3** (PPO, SAC) for ergonomics, or
  **CleanRL** if you want to read the algorithm. **rl_games** if you go
  Isaac.
- Observation: LiDAR scan (downsampled to 108–1080 beams), velocity, last
  action. Avoid raw camera pixels until everything else works.
- Action: continuous `(steering, throttle)` clipped to vehicle limits.
- Reward: progress along centerline minus collision penalty minus jerk
  penalty. Reward shaping is where most of the engineering time goes.
- Tricks that matter: action smoothing, frame stacking, domain randomization
  over friction / mass / latency / sensor noise.

**Compute budget:** PPO on F1TENTH Gym converges in ~10M steps,
30 min – 2 hr on a single GPU. Isaac Lab parallel envs cut that to single-digit
minutes but you'll lose that time to setup.

---

## Phase 6 — Sim2Real (2–4 weeks)

This is where most projects die. Plan for it from day one.

- **Domain randomization**: randomize mass ±20%, friction ±30%, motor delay
  0–80ms, LiDAR noise, IMU bias, wheelbase ±2%. Train across the distribution.
- **System identification**: log real telemetry, fit sim parameters to it,
  retrain in the narrowed distribution.
- **Latency matching**: measure real sensor→action latency and bake the same
  delay into sim.
- **Safety controller**: a simple "if obstacle within X m → brake" wrapper
  around the policy for early on-car runs.
- **Gradual rollout**: empty room → cones at 30% target speed → full track
  at 50% → full track at 100%.

**Useful libraries:** `ros2_control`, `MicroXRCEDDS` (if you go ESP32 for
low-level control), `foxglove-studio` for telemetry visualization.

---

## Phase 7 — Build the physical car (4–8 weeks, parallelizable with Phase 5)

**Bill of materials** (F1TENTH-style 1/10):

| Item | Typical part | ~Cost (USD) |
|---|---|---|
| Chassis + drivetrain | Traxxas Slash 4x4 or Ford Fiesta ST Rally | 350 |
| Motor / ESC | Stock Velineon 3500 or VESC 6 MkVI | 0–250 |
| Compute | Jetson Orin Nano 8GB devkit | 500 |
| LiDAR | Hokuyo UST-10LX (used) or RPLIDAR S2 | 400–1500 |
| Camera | Intel RealSense D435i | 350 |
| IMU | VectorNav VN-100 (or BNO085 for cheap) | 30–800 |
| Power | 3S/4S LiPo + UBEC for compute | 100 |
| Microcontroller | Teensy 4.1 or VESC's onboard | 30 |
| Custom bodywork | 3D-printed PETG/ASA from your CAD | 50 |
| **Total** | | **~$2k–4k** |

**Process:**
1. Mechanical assembly + sensor mounts (CAD'd, 3D printed).
2. Bring up low-level: ESC + steering servo controlled from Teensy/VESC over
   serial, exposed as a ROS 2 topic.
3. Sensor bring-up one at a time — verify each in `rviz2` before integrating.
4. Static safety checks: kill switch, current-limited PSU bench tests.
5. Hand-driven data collection lap → SLAM map → first autonomous lap on the
   classical baseline.
6. Deploy the trained RL policy with the safety wrapper.

---

## Phase 8 — Iterate (ongoing)

- Keep a **shared track description** between sim and real so a sim
  improvement → real test is a one-command deploy.
- Log every real-world lap to a Foxglove `mcap`. Replay them in sim to find
  divergence.
- Re-fit aero coefficients once the bodywork is finalized; refresh the
  vehicle model.
- Re-train policy quarterly or whenever the car is mechanically changed.

---

## Recommended starting toolchain (concrete)

- **Sim:** F1TENTH Gym → Isaac Lab
- **CAD:** Fusion 360
- **CFD:** OpenFOAM in Docker, ParaView for viz
- **Track authoring:** Blender + procedural Python generator
- **RL:** Stable-Baselines3 (PPO) → rl_games on Isaac
- **Robotics middleware:** ROS 2 Jazzy
- **Telemetry:** Foxglove Studio + `mcap` logs
- **Compute on car:** Jetson Orin Nano 8GB
- **Version control / CI:** this repo + GitHub Actions for sim regression
  tests on PR

---

## Suggested milestones

| Week | Milestone |
|---|---|
| 2 | Spec frozen, simulator chosen, repo scaffolded |
| 4 | Procedural track generator + F1TENTH Gym integration |
| 6 | Pure Pursuit baseline laps in sim |
| 8 | Vehicle dynamics validated against bench data |
| 10 | First aero CFD sweep, coefficients in sim |
| 12 | PPO policy beats Pure Pursuit baseline in sim |
| 14 | Physical car assembled, hand-driven |
| 18 | First autonomous lap on classical stack |
| 22 | Sim2real RL policy completes 5 clean laps |
| 26 | Lap-time targets met; write-up |
