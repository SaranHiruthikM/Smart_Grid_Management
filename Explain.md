# Smart Grid Management: Project Explanation and Demo Guide

## 1. What is this project?

This project is a **decentralized smart-grid multi-agent system (MAS)** for
simulating a small residential microgrid.

Each household is represented by an autonomous `HouseholdAgent`. A household
has:

- local solar generation;
- local electricity demand;
- a battery with a state of charge (SOC);
- its own charging and discharging limits;
- its own peer-to-peer (P2P) buying and selling preferences.

The simulation runs in discrete hourly steps. Agents first use their own solar
and battery, then communicate with nearby households to trade energy, and
finally use the external utility grid for any remaining imbalance. The utility
grid is an energy source/sink only; it does not make decisions for the
households.

The main implementation is in [`smart_grid_mas/`](./smart_grid_mas/).

## 2. What problem does it solve?

Traditional centralized energy management can become difficult when many
homes have their own solar panels, batteries, and changing loads. A central
controller would need to collect private household information, make all
decisions, and coordinate every energy transfer. This creates:

- a single point of failure;
- communication and scalability bottlenecks;
- less privacy for households;
- poor handling of intermittent solar generation and changing demand;
- unnecessary imports from the main grid when a neighboring home has surplus.

This project investigates a decentralized alternative:

1. **Each household makes its own local decision.**
2. **Households communicate through structured messages**, rather than sharing
   all private state.
3. **Surplus households can sell energy to deficit households** through a
   Contract-Net-style negotiation.
4. **Batteries can be planned ahead** using local A* search and time-of-use
   tariffs.
5. **The microgrid can be tested under shocks**, including clouds, peak
   demand, and a utility-grid blackout.

This is a simulation and demonstration system. It does not control real
electrical hardware.

## 3. How the solution works

### 3.1 Energy priority at each household

The agent follows this general priority:

1. use local solar directly for local demand;
2. charge or discharge the local battery;
3. negotiate with peer households;
4. import from or export to the utility grid if it is available.

If the utility grid is offline, the grid cannot supply or absorb energy. The
microgrid then operates in islanded mode using solar, batteries, and P2P
trades. Any demand that still cannot be served is reported as unmet demand
instead of being hidden.

### 3.2 Local state classification

For each agent:

```text
energy balance = solar generation - household demand
```

- positive balance: `SURPLUS`;
- negative balance: `DEFICIT`;
- zero balance: `BALANCED`.

The local decision logic is implemented in
[`HouseholdAgent`](./smart_grid_mas/agents/household_agent.py). Agents do not
select a neighborhood-wide winner or receive a centrally assigned action.

### 3.3 P2P negotiation

The model transports structured messages defined in
[`communication/message.py`](./smart_grid_mas/communication/message.py).
Typical message types are:

```text
SURPLUS -> DEFICIT -> REQUEST -> OFFER -> ACCEPT/REJECT
```

When a trade is accepted, it is stored in the model's transaction ledger and
included in telemetry. This makes it possible to show exactly which household
sold energy, which household bought it, the quantity, and the price.

The 11-phase simulation lifecycle in
[`environment/smart_grid_model.py`](./smart_grid_mas/environment/smart_grid_model.py)
separates observation, message exchange, negotiation, settlement, battery
updates, grid fallback, and telemetry collection.

### 3.4 Local A* planning

The Greedy mode reacts immediately to the current hour. The A* mode gives each
household its own look-ahead planner from
[`planning/astar_planner.py`](./smart_grid_mas/planning/astar_planner.py).

The planner:

- looks ahead over a configurable horizon, normally 12 hours;
- considers solar and demand forecasts;
- considers the battery SOC and charge/discharge limits;
- uses time-of-use import tariffs;
- accounts for grid availability, export, cycling, and unmet demand;
- commits only the first action, then replans at the next step.

This is **decentralized planning**: every household plans its own battery
trajectory. A* is not a central optimizer for the whole neighborhood.

## 4. Important project components

| Component | Purpose |
|---|---|
| [`agents/household_agent.py`](./smart_grid_mas/agents/household_agent.py) | Autonomous household behavior, battery actions, P2P participation, and local planning integration |
| [`environment/smart_grid_model.py`](./smart_grid_mas/environment/smart_grid_model.py) | Creates agents, connects neighbors, advances the 11-phase timestep, and records telemetry |
| [`environment/utility_grid.py`](./smart_grid_mas/environment/utility_grid.py) | External grid import/export and blackout availability |
| [`communication/message.py`](./smart_grid_mas/communication/message.py) | Structured inter-agent messages |
| [`negotiation/negotiation_protocol.py`](./smart_grid_mas/negotiation/negotiation_protocol.py) | P2P request, offer, acceptance, and settlement behavior |
| [`planning/astar_planner.py`](./smart_grid_mas/planning/astar_planner.py) | Per-household predictive battery planning |
| [`planning/cost_function.py`](./smart_grid_mas/planning/cost_function.py) | Tariff, battery, export, and unmet-demand cost model |
| [`scenarios/scenario_runner.py`](./smart_grid_mas/scenarios/scenario_runner.py) | 24-hour simulations and resilience metrics |
| [`scenarios/profiles.py`](./smart_grid_mas/scenarios/profiles.py) | Solar, demand, and empirical dataset inputs |
| [`scenarios/shocks.py`](./smart_grid_mas/scenarios/shocks.py) | Cloud, peak-demand, and blackout events |
| [`run_simulation.py`](./run_simulation.py) | One-step P2P trace and before/after P2P comparison |
| [`run_scenario.py`](./run_scenario.py) | 24-hour scenarios and scenario comparison |
| [`run_planning.py`](./run_planning.py) | A* trace and Greedy-vs-A* benchmark |
| [`dashboard.html`](./dashboard.html) | Browser-based SCADA-style visualization |
| [`export_dashboard_data.py`](./export_dashboard_data.py) | Generates simulation-backed dashboard JSON |
| [`generate_dashboard_js.py`](./generate_dashboard_js.py) | Converts JSON into browser data |
| [`smart_grid_mas/tests/`](./smart_grid_mas/tests/) | Automated correctness, negotiation, scenario, planning, and dashboard tests |

## 5. Setup

Run all commands from the project root (the directory containing
`requirements.txt`).

### Linux/macOS

```bash
cd /path/to/Smart_Grid_Management
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

### Windows PowerShell

```powershell
cd path\to\Smart_Grid_Management
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

The required Python packages are listed in
[`requirements.txt`](./requirements.txt). The code uses Mesa for the
agent-based simulation, NetworkX for graph-related support, and Pytest for
validation.

## 6. Run the agents independently and demonstrate the solution

There is no separate server process for each agent. The normal design is one
Python process containing multiple independent `HouseholdAgent` objects. Each
agent owns its state and decisions, while `SmartGridModel` only provides the
environment, message transport, and bookkeeping.

The following demos isolate the capabilities so that each problem can be
shown independently.

### Demo A: One-step P2P negotiation

Run the before/after comparison:

```bash
python run_simulation.py --compare
```

This runs the same deterministic three-household setup twice:

- P2P disabled: the deficit household uses the utility grid;
- P2P enabled: the surplus household offers energy to the deficit household.

Look for these results:

- P2P energy traded changes from zero to a positive value;
- P2P trade count and trade value become positive;
- utility-grid import decreases;
- the output concludes that external grid import was reduced.

To see one negotiation trace rather than the comparison:

```bash
python run_simulation.py --steps 1
```

The output shows the agent states, `REQUEST`, `OFFER`, acceptance reason, trade
quantity and price, remaining energy imbalance, and grid import/export.

To run several independent simulation steps:

```bash
python run_simulation.py --steps 5
```

To scale the neighborhood without changing the model code:

```bash
python run_simulation.py --households 10 --steps 2
```

To explicitly run the no-P2P baseline:

```bash
python run_simulation.py --no-p2p --steps 1
```

### Demo B: 24-hour normal operation and resilience shocks

Run one complete day under the combined stress test:

```bash
python run_scenario.py --scenario combined
```

The scenario runner uses 24 hourly steps and reports solar generation, demand,
P2P energy, grid import/export, unmet demand, battery SOC, and active events.
The combined scenario applies:

- cloud cover from 12:00 to 13:00, reducing solar;
- peak demand from 17:00 to 19:00, increasing load;
- a grid outage from 18:00 to 19:00, forcing islanded operation.

Run the individual shocks independently:

```bash
python run_scenario.py --scenario baseline
python run_scenario.py --scenario cloud_cover
python run_scenario.py --scenario peak_demand
python run_scenario.py --scenario grid_outage
python run_scenario.py --scenario real_world_dataset
```

What demonstrates that the scenarios work:

- `cloud_cover` shows lower solar during the event hours;
- `peak_demand` shows higher demand during the evening event;
- `grid_outage` shows `GRID BLACKOUT`, zero grid import while islanded, and
  any unavoidable unmet demand explicitly reported;
- `real_world_dataset` uses
  [`data/benchmark_microgrid_dataset.csv`](./data/benchmark_microgrid_dataset.csv);
- `combined` exercises all mechanisms in one run.

Run all standard analytical scenarios in one comparison table:

```bash
python run_scenario.py --compare-all
```

This is the best terminal demonstration for resilience because it puts the
same metrics side by side for baseline, cloud cover, peak demand, blackout,
and combined stress.

### Demo C: A* planning versus reactive Greedy control

Run the 24-hour benchmark:

```bash
python run_planning.py --compare --scenario baseline
```

Run the benchmark under a difficult scenario:

```bash
python run_planning.py --compare --scenario combined --horizon 12
```

Compare these columns:

- total grid import;
- peak-hour import from 17:00 to 21:00;
- unmet demand;
- average battery SOC;
- total time-of-use electricity cost.

A* is helping when its cost, peak import, or grid import is lower than the
Greedy result without creating an unacceptable increase in unmet demand. The
program calculates the verdict from the actual numbers, so do not assume that
A* must win every metric in every scenario.

Show one individual household's search trace:

```bash
python run_planning.py --trace --horizon 6
```

This displays the forecast, tariff for each hour, number of expanded search
nodes, computation time, selected action sequence, battery flow, SOC, import,
export, and the first receding-horizon action committed.

Study the effect of planning horizon:

```bash
python run_planning.py --horizon-study --scenario baseline
```

### Demo D: Browser dashboard

Generate fresh dashboard data from the Python simulation:

```bash
python export_dashboard_data.py
python generate_dashboard_js.py
```

Then serve the directory locally:

```bash
python -m http.server 8000
```

Open [http://localhost:8000/dashboard.html](http://localhost:8000/dashboard.html)
in a browser. The dashboard compares Greedy and A* modes, displays hourly
solar/demand/grid/P2P telemetry, shows household nodes and their messages, and
exposes shock hours.

The dashboard is static HTML, but its values are generated from the Python
simulation. If the scenario code changes, regenerate
`dashboard_data.json` and `dashboard_data.js` before presenting the dashboard.

Stop the local server with `Ctrl+C`.

## 7. Run the automated proof

Run the full test suite:

```bash
pytest smart_grid_mas/tests/ -v
```

Or use Python's built-in unittest discovery:

```bash
python -m unittest discover -s smart_grid_mas/tests -p "test_*.py"
```

The tests cover:

- local energy states and battery bounds;
- P2P message and trade behavior;
- energy conservation and grid accounting;
- cloud, peak-demand, outage, combined, and dataset scenarios;
- A* planning optimality and planning integration;
- dashboard data generation and field structure;
- multi-household scalability.

Tests are the repeatable proof that the implementation preserves its
invariants. The CLI demos are the human-readable proof that makes the
behavior easy to explain.

## 8. Suggested presentation order

For a short live demonstration:

1. Explain the problem: local solar and demand do not match at every home or
   every hour.
2. Run `python run_simulation.py --compare` to show that P2P sharing reduces
   external-grid dependence.
3. Run `python run_scenario.py --scenario combined` to show clouds, peak
   demand, and islanding.
4. Run `python run_planning.py --compare --scenario combined` to compare
   reactive Greedy control with predictive local A* planning.
5. Open the regenerated dashboard to visualize the same telemetry.
6. Finish with `pytest smart_grid_mas/tests/ -v` to show automated validation.

When explaining the architecture, emphasize that the household agents are
independent decision-makers. The model coordinates time and records results;
it does not centrally assign energy trades or battery actions.

## 9. Interpreting the most important metrics

| Metric | Meaning | Desired direction |
|---|---|---|
| P2P Energy Traded | Energy transferred between households | Higher when local sharing is available |
| Grid Import | Energy purchased from the utility grid | Lower |
| Grid Export | Surplus sent to the utility grid | Context-dependent; lower can mean more local use |
| Unmet Demand | Demand not served, especially during islanding | Lower |
| P2P Deficit Coverage | Portion of deficit served by peers | Higher |
| Average/Minimum SOC | Battery energy reserve over the run | Avoid unsafe depletion; scenario-dependent |
| Total Energy Cost | Time-of-use import cost plus modeled costs | Lower |
| Peak-Hour Import | Grid dependence during expensive evening hours | Lower |
| Outage Duration | Hours for which the utility grid is unavailable | Event input, not a control result |

For a fair comparison, keep the scenario, number of households, and random
seed the same between runs. The provided CLI defaults use three households and
seed `42` where applicable.
