# Development Plan — Simulation & RL Stack

A near-term, code-focused plan for the software half of the project. The
[`roadmap.md`](./roadmap.md) covers the full sim-to-real arc; this document
zooms in on the four pieces we build *now*:

1. **Track building** — author and procedurally generate drivable tracks.
2. **Air flow simulation** — produce aero coefficients that feed the vehicle model.
3. **RL pipeline** — training, logging, evaluation, checkpointing.
4. **Gymnasium-compatible environment** — the contract that ties it all together.

The ordering matters: the **environment is the spine**. Track building feeds its
reset, aero feeds its dynamics, and the RL pipeline consumes its `step`/`reset`.
So we stand up a thin vertical slice of the env first, then deepen each pillar.

---

## Target repository layout

```
downforce-robotics/
├── docs/
│   ├── roadmap.md                 # existing strategic roadmap
│   └── development-plan.md         # this file
├── src/downforce/
│   ├── __init__.py
│   ├── envs/
│   │   ├── __init__.py            # Gymnasium registration
│   │   ├── racing_env.py         # RacingEnv(gymnasium.Env)
│   │   └── wrappers.py           # action smoothing, frame stack, normalization
│   ├── track/
│   │   ├── __init__.py
│   │   ├── spec.py               # Track dataclass + (de)serialization
│   │   ├── generator.py          # procedural centerline generation
│   │   ├── mesh.py               # centerline -> drivable mesh / boundaries
│   │   └── raceline.py           # min-curvature optimal line (reference)
│   ├── dynamics/
│   │   ├── __init__.py
│   │   ├── kinematic_bicycle.py
│   │   ├── dynamic_bicycle.py    # + Pacejka tire
│   │   └── aero.py               # speed/yaw-dependent aero force model
│   ├── aero/
│   │   ├── README.md             # CFD workflow + coefficient table format
│   │   ├── cases/                # OpenFOAM case templates
│   │   └── coefficients.csv      # Cd, Cl_f, Cl_r, Cy vs speed & yaw
│   ├── rl/
│   │   ├── train.py              # SB3 PPO/SAC entrypoint
│   │   ├── evaluate.py           # rollout, metrics, video
│   │   ├── configs/              # YAML hyperparameters
│   │   └── rewards.py            # reward terms (progress, collision, jerk)
│   └── utils/
│       ├── geometry.py           # splines, frenet transforms
│       └── logging.py            # telemetry / mcap helpers
├── tests/
│   ├── test_track.py
│   ├── test_env_api.py           # gymnasium.utils.env_checker
│   └── test_dynamics.py
├── notebooks/                    # exploration, raceline viz, aero plots
├── pyproject.toml
└── requirements.txt
```

---

## Pillar 1 — Track building

**Goal:** one canonical track description that everything else derives from.

### Canonical format (`track/spec.py`)
A `Track` dataclass holding per-waypoint centerline data:
`(x, y, width_left, width_right, banking)`, plus metadata (name, closed loop,
units). Serialize to/from YAML and CSV so tracks are diffable and shareable
between sim and (later) real. This is the single source of truth — mesh,
occupancy grid, and raceline are all *derived*.

### Procedural generation (`track/generator.py`)
- Sample N random control points on a ring, enforce a minimum spacing and a
  maximum curvature so tracks stay drivable.
- Fit a **periodic Catmull-Rom / cubic spline** through them for a smooth
  closed centerline.
- Assign track width (constant first, then varied) → produces left/right
  boundaries via normal offset.
- Seedable RNG so a track is reproducible from an integer. This is what gives
  us *infinite randomized tracks* for generalization.

### Mesh & boundaries (`track/mesh.py`)
- Extrude the centerline to a flat drivable surface with `trimesh`.
- Emit boundary polylines (for collision / LiDAR raycasting) and an occupancy
  grid (for localization later).

### Optimal raceline (`track/raceline.py`)
- Offline **minimum-curvature** optimization (`casadi` or a QP) to get a
  reference line + speed profile. Used for reward shaping and as a baseline
  to beat. Not on the hot path.

**Deliverable:** `Track.random(seed=…)` returns a valid track; round-trips
through YAML; renders centerline + boundaries in a notebook.

---

## Pillar 2 — Air flow simulation

**Goal:** turn bodywork geometry into a coefficient table the sim can read
cheaply. **CFD never runs inside the RL loop** — it's ~10⁶× too slow. We run it
offline and tabulate.

### Workflow (`aero/README.md`)
1. CAD the body (Fusion 360 / OnShape) → export STL.
2. Mesh with `snappyHexMesh` / `cfMesh`.
3. Steady-state RANS (`simpleFoam`, OpenFOAM in Docker) across a sweep:
   speeds `{5, 10, 15} m/s` × yaw `{0°, 5°, 10°}` → 9 runs.
4. Extract `Cd, Cl_front, Cl_rear, Cy`; visualize in ParaView.
5. Write results to `aero/coefficients.csv`.

### Coefficient table format (`aero/coefficients.csv`)
```
speed_mps, yaw_deg, Cd, Cl_front, Cl_rear, Cy, frontal_area_m2
```

### Consumption in sim (`dynamics/aero.py`)
- Load the CSV, **interpolate** (bilinear over speed × yaw).
- Convert coefficients → forces: `F = ½·ρ·v²·A·C`.
- Apply downforce as added normal load on front/rear axles (raises the tire
  grip limit), drag opposing motion, side force at yaw.
- Provide a **null/constant model** so the env runs before any CFD exists.

**Deliverable:** `AeroModel(coefficients.csv)` returns aero forces given
`(v, yaw)`; falls back gracefully to zero-aero. CFD cases are templated but can
be run later — the interface is what unblocks Pillar 4.

---

## Pillar 3 — Gymnasium-compatible environment

**This is the spine — build a thin version first.**

### Contract (`envs/racing_env.py`)
- Subclass `gymnasium.Env`; implement `reset(seed, options)` and `step(action)`
  with the **5-tuple** return `(obs, reward, terminated, truncated, info)`.
- Declare `observation_space` and `action_space` as `gymnasium.spaces.Box`.
- Set `metadata = {"render_modes": ["rgb_array", "human"], "render_fps": …}`.
- **Observation:** simulated LiDAR scan (downsampled, e.g. 108 beams via
  raycast against track boundaries) + body-frame velocity + last action.
- **Action:** continuous `[steering, throttle]`, clipped to vehicle limits.
- **Termination:** crossing a boundary (collision) or lap complete.
  **Truncation:** max episode steps.
- **Reward:** delegated to `rl/rewards.py` (progress − collision − jerk).

### Internals
- `reset` pulls a `Track` (fixed or `Track.random(seed)` for domain
  randomization), resets the dynamics state to the start line.
- `step` integrates the chosen dynamics model (`kinematic` → `dynamic_bicycle`)
  with the aero force injected, advances time, raycasts the new scan,
  evaluates reward and termination.
- `render` draws track, car pose, and scan (matplotlib → rgb_array first).

### Registration & wrappers
- Register ids in `envs/__init__.py`:
  `Downforce-Racing-v0` (kinematic), `Downforce-Racing-Dynamic-v0`.
- `wrappers.py`: action smoothing, frame stacking, observation normalization,
  time-limit (or use Gymnasium's built-ins).

### Conformance — non-negotiable
- `tests/test_env_api.py` runs `gymnasium.utils.env_checker.check_env` and a
  random-agent smoke rollout in CI. If the env isn't spec-compliant, SB3 and
  every other tool silently misbehave, so this gate comes first.

**Deliverable:** `gymnasium.make("Downforce-Racing-v0")` passes `check_env`
and survives 1000 random steps without error.

---

## Pillar 4 — RL pipeline

**Goal:** reproducible train → evaluate → checkpoint loop. Baseline before RL.

### Classical baseline first (`rl/` or `dynamics/`)
- **Pure Pursuit** following the precomputed raceline. Sets the bar and doubles
  as the safety controller for real-world testing. RL must beat this number.

### Training (`rl/train.py`)
- **Stable-Baselines3** PPO (start) / SAC (continuous, sample-efficient).
- `SubprocVecEnv` for parallel envs; `VecNormalize` for obs/reward scaling.
- Hyperparameters in `rl/configs/*.yaml` — no magic numbers in code.
- Checkpoints + `VecNormalize` stats saved together; resumable.

### Reward design (`rl/rewards.py`)
- Centerline/raceline **progress** (primary), **collision** penalty,
  **jerk/steering-rate** penalty for smoothness. Each term individually
  toggleable and logged so we can see which dominates.

### Domain randomization
- Per-episode randomize: track (seed), friction ±30%, mass ±20%, actuator
  latency 0–80 ms, sensor noise. This is the single biggest lever for eventual
  sim2real, so it's wired into `reset` from the start, gated by config.

### Evaluation (`rl/evaluate.py`)
- Deterministic rollouts on a **held-out track set** (seeds unseen in
  training). Metrics: lap time, completion rate, collisions, mean speed.
- Render `rgb_array` rollout videos as artifacts.

### Experiment tracking
- TensorBoard locally; structure for an optional Weights & Biases hook.
- Log to `mcap`/Foxglove-friendly format to keep parity with the real-car
  telemetry path described in the roadmap.

**Deliverable:** `python -m downforce.rl.train --config configs/ppo.yaml`
trains a PPO policy that completes laps and beats Pure Pursuit lap time on the
held-out set; `evaluate.py` emits metrics + a video.

---

## Execution order & milestones

Each milestone is an independently useful, testable artifact.

| # | Milestone | Pillars | Definition of done |
|---|---|---|---|
| M0 | Repo scaffold | — | `pyproject.toml`, package layout, CI runs `pytest` + lint |
| M1 | Track spec + procedural generator | Track | `Track.random(seed)` round-trips YAML, renders; `test_track.py` green |
| M2 | Thin Gym env (kinematic, zero aero) | Env | `check_env` passes; 1000 random steps; registered id |
| M3 | RL smoke train | RL + Env | PPO runs end-to-end, reward trends up, checkpoint saved |
| M4 | Dynamic bicycle + Pacejka + aero interface | Dynamics + Aero | trajectory test vs reference; null-aero default |
| M5 | Aero coefficients wired in | Aero | `coefficients.csv` consumed; downforce shifts grip limit measurably |
| M6 | Pure Pursuit baseline + eval harness | RL + Track | baseline laps held-out tracks; `evaluate.py` metrics + video |
| M7 | Domain randomization + PPO beats baseline | RL | PPO > Pure Pursuit lap time on unseen tracks |

**Suggested first two weeks:** M0 → M1 → M2 → M3. That gives a registered,
spec-compliant Gymnasium env training a policy on procedurally generated tracks
— the smallest thing that proves the architecture, with dynamics fidelity and
aero layered on afterward without touching the env contract.

---

## Tooling & dependencies

- **Core:** `gymnasium`, `numpy`, `scipy`, `pyyaml`
- **Track/geometry:** `trimesh`, `shapely`, `casadi` (raceline)
- **RL:** `stable-baselines3`, `torch`, `tensorboard`
- **Aero (offline):** OpenFOAM (Docker), ParaView; `pandas` to read the table
- **Dev:** `pytest`, `ruff`, `mypy` (optional), GitHub Actions for CI

## CI gates

- `env_checker` conformance test on every PR — the env contract never regresses.
- Track generation property tests (closed loop, max-curvature bound, valid widths).
- A short headless training smoke run (few thousand steps) to catch breakage in
  the RL entrypoint without burning CI minutes.
</content>
</invoke>
