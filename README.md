# ChocoSim

A physically motivated simulator of chocolate manufacturing, from raw bean to
packed bar. It models the unit operations, the production line that contends
for them, the quality that comes out, the money it costs, and the optimisation
problems that sit on top.

> **Read this first.** Every output is **SIMULATION** grade. The models are
> built on real physical relationships — Arrhenius kinetics, Casson rheology,
> cocoa butter polymorphism, first-law energy balances — but their rate
> constants are literature-scale values and internal calibrations, not
> measurements from any plant. Nothing here has been validated against a
> physical factory. See [Fidelity](#fidelity) below.

---

## What it does

- **Bean to bar.** Roasting, winnowing, grinding, mixing, refining, conching,
  tempering, moulding, cooling, packing — each a real model with its own
  kinetics, energy balance and mass balance.
- **Cocoa butter polymorphism.** Six crystal forms with distinct melting
  points, nucleation rates and stability. The tempering cycle is simulated as
  an ODE over the polymorph distribution, and the model reproduces the narrow
  reheat window that makes tempering difficult in practice.
- **Discrete-event production line.** SimPy model with sequence-dependent
  changeovers, Weibull machine failures and parallel units, measuring where
  work actually queues rather than assuming a bottleneck.
- **Quality scoring.** A 0–100 batch score where every point lost is
  attributable to a named metric, and every defect to a named cause.
- **Economics.** Bottom-up absorption costing, unit economics per bar, and a
  period profit statement.
- **Optimisation** via OR-Tools: CP-SAT sequencing, LP reformulation, MILP
  energy load shifting, batch sizing.
- **Scenarios and Monte Carlo.** "What if cocoa rises 20%" answered with a
  number, plus P10/P50/P90 bands from replicated runs.
- **Interfaces.** A FastAPI service, a CLI, a console status panel and a
  self-contained HTML dashboard.

## Install

```bash
pip install -e ".[dev]"
```

Python 3.12+. Dependencies: pydantic, numpy, scipy, polars, pandas, simpy,
ortools, fastapi, sqlalchemy, plotly, uvicorn.

## Quick start

```bash
# What is in the recipe book?
chocosim recipes

# Full analysis of one product
chocosim recipe RCP-DARK-70

# Produce a batch: process report, quality certificate, costing sheet
chocosim batch RCP-DARK-70 --batch-kg 1500

# Compare roast profiles, and find the equal-development time at 165 C
chocosim roast --equivalent-at 165

# Where does the line actually bottleneck?
chocosim line --batches 40 --days 14

# Optimise the production sequence
chocosim schedule --batches 12

# Current versus optimised
chocosim optimise

# What if cocoa, sugar, energy and labour all move?
chocosim scenario

# P10/P50/P90 across replications
chocosim montecarlo --replications 50

# Least-cost reformulation that keeps the product the same product
chocosim reformulate RCP-MILK-35

# Dashboards
chocosim dashboard                       # console panel
chocosim dashboard -o dashboard.html     # HTML with charts

# API
chocosim serve
```

Or as a library:

```python
import datetime as dt
from chocosim.core.config import ChocoSimConfig
from chocosim.production.factory import Factory

factory = Factory(ChocoSimConfig.default().deterministic())
factory.stock_up(dt.date.today(), days_of_cover=45.0, daily_demand_kg=10_000)
outcome = factory.produce_batch("RCP-DARK-70", 1500.0)

print(outcome.process.report())
print(outcome.quality.report())
print(outcome.costing.report())
```

The end-to-end demonstration runs every subsystem in one pass:

```bash
python -m examples.full_factory_run
```

## Module map

| Package | What lives there |
|---|---|
| `core` | Mass balance, composition algebra, units, seeded RNG, configuration, **fidelity registry** |
| `ingredients` | 22-ingredient catalogue, 7 suppliers, lot tracking with FEFO issue, price shocks |
| `recipes` | Versioned recipes, costing, composition, GB compositional standards, nutrition |
| `processing` | Roasting, winnowing, grinding, mixing, refining, conching, moulding, cooling, rheology, and the chain that sequences them |
| `tempering` | Cocoa butter polymorphs, nucleation/growth/ripening kinetics, bloom risk |
| `quality` | Metric scoring, defect inference, batch disposition |
| `production` | Machines, SimPy line simulation, factory digital twin |
| `packaging` | Pack formats, give-away model, line performance |
| `inventory` | Raw/WIP/finished stock, reorder policy with safety stock |
| `logistics` | Supplier performance, lead-time variance, disruption events |
| `energy` | Duty calculation by carrier, time-of-use tariffs, carbon |
| `economics` | Absorption costing, unit economics, profit and loss |
| `optimisation` | CP-SAT sequencing, LP blending, MILP energy shifting, factory tuning |
| `simulation` | What-if scenarios, Monte Carlo |
| `telemetry` | Synthetic sensors with noise, drift and faults |
| `api` | FastAPI service |
| `dashboard` | Console panels, HTML report |

## Fidelity

The specification asked for a strict separation between simulation, prediction
and engineering validation, and this is enforced in code rather than in
comments. `chocosim.core.fidelity` defines three levels:

| Level | Requires | Status in this repo |
|---|---|---|
| `SIMULATION` | Nothing | **Everything is here** |
| `PREDICTION` | A `CalibrationRecord` with a dataset reference, sample count and residual RMSE | Nothing reaches this |
| `ENGINEERING_VALIDATION` | Prior calibration plus a `ValidationRecord` naming a responsible engineer and evidence | Nothing reaches this |

Constructing a `ModelProvenance` that claims `PREDICTION` without a
calibration record raises `FidelityError`. This is tested, and it caught a
real mistake during development: the Monte Carlo engine initially claimed
`PREDICTION`, which is wrong — running an uncalibrated model a thousand times
gives you a very precise description of the model and tells you nothing new
about the factory.

Every report, API response and dashboard carries its fidelity banner, so a
number cannot escape the process without its epistemic status attached.

**What these outputs are good for:** comparing process options against each
other, exploring trade-offs, sizing the direction and rough magnitude of a
change, and building intuition about where a chocolate factory's constraints
and costs actually sit.

**What they are not good for:** setting process parameters on real equipment,
food safety decisions, regulatory submissions, or substituting for laboratory
analysis and sensory panels.

### Calibration table

Where a rate constant was not available from first principles, it was fitted
numerically so that a named reference case reproduces a known qualitative
behaviour. These are internal calibrations against *intended model behaviour*,
not against plant data.

| Constant | Value | Anchored so that |
|---|---|---|
| `FLAVOUR_PREEXPONENTIAL` | 4.855446e8 | Roast B (145 °C, 35 min) gives development ratio 1.000 |
| `DAMAGE_PREEXPONENTIAL` | 3.071957e12 | Roast B gives damage ratio 0.20 |
| `CONCHE_DRYING_PREEXPONENTIAL` | 3.85e4 | A dark conche at 72 °C is ~90% dried after 9 h |
| `YIELD_VALUE_SCALE` | 600 | Standard formulations land in the 2–5 Pa band at 40 °C |
| `PHI_MAX_BASE` / `INTRINSIC_VISCOSITY` | 0.600 / 3.5 | Krieger-Dougherty gives workable viscosities across the recipe book |
| `TEMPER_INDEX_REFERENCE_SOLIDS` | 0.070 | 3.5% solid fat of pure Form V reads exactly 5.0 on a temper meter |

## Verified behaviours

These are model outputs, reproduced by the test suite:

- **Roast trade-off.** Matching Roast B's flavour development at 165 °C takes
  11.4 minutes instead of 35, but damage rises from 0.20 to 0.32 and the
  quality index falls from 83.0 to 72.9. Hotter and faster is not free.
- **Tempering window.** For dark chocolate the acceptable reheat window is
  31.0–32.0 °C. At 30 °C the mass is over-seeded; at 33 °C the Form V seed
  melts out and purity collapses to 65%.
- **Conching returns.** Marginal quality per hour falls roughly 400× between
  hour 4 and hour 36 while energy climbs linearly.
- **Bean-to-bar yield.** A recipe starting from whole beans yields 84.8%
  against 97.5% from bought-in liquor — the shell is 12.3% of the input.
- **Sequencing.** Campaigning like products and paying the allergen clean once
  cut changeover time 70% (17.5 h → 5.25 h) on a 10-batch programme.
- **Give-away.** A 100 g bar filled to a 2.5% underweight tolerance with 1.1%
  filling sigma over-fills by 2.16%, which is £690k a year at 40M units.
- **Cocoa exposure.** A 20% cocoa price rise moves unit cost +12.6% on a dark
  product mix; dark is materially more exposed than milk.

## Known limitations

- **No calibration.** The single most important limitation. See above.
- **Thermally lumped.** Bars, beans and conche charges are treated as
  isothermal. No internal gradients are resolved, so surface-versus-core
  effects during cooling are approximated by a quench-rate penalty rather than
  computed.
- **Sensory is inferred, not measured.** Quality scores come from instrument
  values via a weighted rubric. No panel data underlies them, and the weights
  are a commercial judgement encoded in `QualityConfig`.
- **Compositional standards are GB/EU only.** US and other regimes differ and
  are not implemented.
- **Reformulation assumes process invariance.** The LP holds process behaviour
  fixed while composition moves, which is only reasonable for small changes.
- **Suppliers are independent.** No correlated failures, which understates
  tail risk in the supply chain model.
- **Scenarios are mechanical.** They show the consequence of a parameter move,
  not what a business would do in response.
- **Monte Carlo bounds the model, not the world.** Percentiles reflect the
  variability the model was given. Real plants have failure modes nobody
  parameterised.
- Python 3.12 was used for development; the specification asked for 3.13+.

## Tests

```bash
pytest chocosim/tests/            # 281 tests
pytest --cov=chocosim             # 89% coverage
```

The suite is deterministic by default. Tests assert real behaviour — mass
conservation across every stage of every recipe, that finer grinding thickens
the mass, that under-temper is penalised harder than over-temper, that a
scenario's price shock does not leak into the next run — rather than
re-stating the implementation.

## Licence

Proprietary. © Aureom.AI.
