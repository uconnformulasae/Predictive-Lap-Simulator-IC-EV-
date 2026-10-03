# Predictive LapSim

A lap simulation for the team's Formula SAE cars (ICE and EV). It predicts lap time, speed, energy use, pack current, state of charge and pack temperature for the acceleration, skidpad, autocross and endurance events.

> **Project Status:** under active development (2026–27 season). The original quasi-steady-state (QSS) simulation is being migrated onto a modular structure without changing its results, and a first time-domain solver is being added and checked against it. Higher-fidelity vehicle models, telemetry replay through the new pipeline, the web app connection (hopefully) and batch computing come next; see [Next steps](#next-steps).

## The Immediate Goal:

- **QSS solver** (`--solver qss`): the calibrated simulation the team has used for the EV Endurance event, now built from separate subsystem modules. Its results match the previous version to 1e-9.
- **Point-mass time-domain solver** (`--solver pointmass`): integrates the car forward in time (with a driver) that follows the QSS speed profile. It agrees with the QSS within 1% on lap time.
- **Validated car configurations** for CT-16 and CT-17 on both the ICE and EV: every parameter is checked for units, bounds and source when a file loads.
- **A parameter register** that lists every parameter with its value, unit, source and confidence.
- **A test suite** that protects the calibrated results and checks the physics against hand calculations.

## Repository layout

```
lapsim/                  Python package
  config/                schema, loader, parameter register
  core/                  integrator, track, events, results
  solvers/               qss.py, pointmass.py
  subsystems/            one module per part of the car
    aero.py  tires.py  driveline.py  brakes.py
    power/
      base.py            what every power module must provide
      ev.py              motor and inverter limits
      battery.py         pack voltage, resistance, limits, state of charge
  validation/            tools that compare two runs
  cli.py                 the `lapsim` command
cars/
  ct16_ev/vehicle.yaml
  ct17_ev/vehicle.yaml
tracks/                  track centrelines and event definitions
tests/                   see "Testing" below
scripts/make_golden.py   regenerates the frozen reference results
docs/                    contracts, decision records, model cards
```

Telemetry and large result files are not stored in git. Keep them in the team's shared data folder under `raw/` (never edited) and `clean/`.

## Installing

Requires Python 3.11 or newer.

```bash
git clone https://github.com/<team-org>/lapsim.git
cd lapsim
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
pytest -m "not slow"
```

If the tests do not pass on a fresh clone, **open an issue** before changing anything.

## Running the sim

```bash
# Check a car file for errors and unused keys
python -m lapsim check-config cars/ct16_ev/vehicle.yaml

# List every parameter with its unit, source and confidence
python -m lapsim register cars/ct16_ev/vehicle.yaml

# Run an event
python -m lapsim run --car cars/ct16_ev/vehicle.yaml --event endurance --solver qss
python -m lapsim run --car cars/ct16_ev/vehicle.yaml --event autocross --solver pointmass

# Change a parameter for one run without editing the file
python -m lapsim run --car cars/ct16_ev/vehicle.yaml --event autocross --solver qss \
    --set powertrain.gear_ratio=3.8

# Choose where results go
python -m lapsim run --car cars/ct16_ev/vehicle.yaml --event acceleration --solver qss --out results/accel_test
```

Each run writes a folder containing the results as a Parquet table and a `metadata.json` holding the full configuration, the git commit and a hash of every input file. That is enough to reproduce any result later.

**Events:** `acceleration`, `skidpad` (with `--direction left` or `right`), `autocross`, `endurance`.

## Car files

Each car is a single YAML file. The format is unchanged from the original simulation; the comments beside each value record where it came from.

```yaml
vehicle:
  mass_kg: 278.0            # DSS 301: 210 kg car + 68 kg driver
  wheelbase_m: 1.549        # DSS
  cg_height_m: 0.2794       # estimate, to be measured
powertrain:
  gear_ratio: 3.6363        # 40/11 teeth, confirmed by telemetry
  motor_speed_max_rpm: 3000 # inverter speed governor
```

Rules:

- Use **SI units**, with the unit at the end of the name (`mass_kg`, `torque_limit_inverter_nm`).
- **NEVER** put a physical constant directly in the code. Add it to the car file and the schema.
- The loader warns about keys it does not use, so typos and leftover values are caught early.

## Testing

Tests live in `tests/` and run with `pytest`.

| Folder | What it checks | Run time |
| --- | --- | --- |
| `tests/unit/` | Each subsystem against hand calculations; config loading and validation; the comparison tool; results saving; the command line | Seconds |
| `tests/verification/` | Integrator accuracy, energy balance, constant-force motion against the exact solution, QSS properties, point-mass vs QSS agreement | About a minute |
| `tests/golden/` | Regression: today's results against frozen reference results | About a minute |

```bash
pytest -m "not slow"        # everyday run, also run by CI on every push
pytest -m slow              # time-step convergence study, run before merging solver changes
pytest tests/unit -v        # one folder, verbose
```

Hand-calculated reference numbers are kept in one place, `tests/reference_values.py`, each with its derivation, so anyone can check them.

## Checking results

A result counts as trustworthy only if it passes two kinds of check.

**Verification: does the code do what the equations say?**

- Every subsystem returns what a hand calculation gives (for example, drag at 80 km/h matches the 431 N measured for the design spec sheet).
- The integrator reaches its expected fourth-order accuracy on problems with exact solutions.
- Energy in equals energy out over a run.
- A constant force produces exactly the speed the equations predict.
- The skidpad gives identical results turning left and right.
- The point-mass solver agrees with the QSS within 1% on lap time.

**Regression: did a change move a calibrated result?**

The golden tests rerun every trusted case and compare it with the frozen reference at every point along the lap. If a change is meant to move a result, regenerate the references and explain why in `CHANGELOG.md`:

```bash
python scripts/make_golden.py --force --reason "Fixed driveline drag sign"
```

Checking the model against real track data (validation) uses the telemetry pipeline, which is the next major piece of work.


## Next steps

- 7-DOF and 14-DOF vehicle models
- AiM telemetry import and replay through the new pipeline
- Connection to the web app
- Measured inertias, CG height and suspension geometry
- Validation against held-out track sessions
