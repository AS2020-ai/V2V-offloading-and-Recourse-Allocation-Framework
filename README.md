# V2V Task Offloading Simulator based on Greedy Algorithm

A Python discrete-event simulator for benchmarking **Vehicle-to-Vehicle (V2V) computational task offloading strategies**. It is an exact translation of the companion HTML/JS simulator, sharing the same RNG stream and physics model for cross-platform reproducibility.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage](#usage)
- [Mobility Modes](#mobility-modes)
- [Offloading Strategies](#offloading-strategies)
- [Configuration Parameters](#configuration-parameters)
- [Output & Charts](#output--charts)
- [Architecture](#architecture)
- [Key Metrics](#key-metrics)

---

## Overview

Each simulation step, vehicles move around a 2-D map, generate computational tasks via a Poisson arrival process, and offload those tasks to neighbors according to a greedy strategy. Tasks are processed FCFS on each vehicle's CPU. The simulator tracks completion delay, deadline satisfaction, CPU utilisation, and communication energy, then produces time-series charts and a summary table for strategy comparison.

---

## Requirements

- Python 3.8+
- `matplotlib`

Install dependencies:

```bash
pip install matplotlib
```

---

## Installation

```bash
git clone <repo-url>
cd v2v-simulator
pip install matplotlib
```

No additional setup is required. The simulator is a single self-contained script.

---

## Quick Start

```bash
# Run all 5 strategies with default synthetic mobility (25 vehicles, 120 s)
python v2v_simulator_final.py

# Load a SUMO FCD XML trace
python v2v_simulator_final.py --trace trace.fcd.xml

# Load a generic CSV trace
python v2v_simulator_final.py --trace mobility.csv

# Run only two specific strategies
python v2v_simulator_final.py --strategies V2V_BASELINE LOCAL_ONLY

# Customise vehicles, duration, and task arrival rate
python v2v_simulator_final.py --N 30 --T 200 --arrival-rate 10 --seed 7

# Run without generating plots
python v2v_simulator_final.py --no-plots
```

---

## Usage

```
python v2v_simulator_final.py [options]
```

### General Options

| Argument | Default | Description |
|---|---|---|
| `--trace PATH` | *(none)* | Path to a SUMO FCD `.xml` or CSV trace file. Omit for synthetic mode. |
| `--strategies S [S ...]` | all 5 | Strategies to run. Choices: `V2V_BASELINE`, `LOCAL_ONLY`, `RANDOM`, `NEAREST`, `ROUND_ROBIN`. |
| `--output-dir DIR` | `v2v_results` | Directory where PNG charts are saved. |
| `--no-plots` | *(flag)* | Skip chart generation; print summary only. |

### Simulation Options

| Argument | Default | Description |
|---|---|---|
| `--N` | 25 | Number of vehicles (synthetic mode only). |
| `--T` | 120.0 | Simulation duration in seconds (overridden by trace duration when a file is loaded). |
| `--dt` | 0.5 | Simulation time step in seconds. |
| `--seed` | 42 | RNG seed for reproducible results. |
| `--comm-range` | 100.0 | V2V communication range in metres. |
| `--map-size` | 500.0 | Side length of the square map in metres (synthetic mode). |
| `--max-speed` | 20.0 | Maximum vehicle speed in m/s (synthetic mode). |

### Task Options

| Argument | Default | Description |
|---|---|---|
| `--arrival-rate` | 8.0 | Mean Poisson task arrival rate (tasks/second across all vehicles). |
| `--ds-min` / `--ds-max` | 100 / 1000 | Task payload size range in bytes. |
| `--cycles-min` / `--cycles-max` | 500 / 2000 | CPU cycle demand range per task. |
| `--dl-min` / `--dl-max` | 2.0 / 6.0 | Task deadline range in seconds (relative to arrival). |
| `--cpu-min` / `--cpu-max` | 0.5 / 1.5 | Per-vehicle CPU frequency bounds in GHz. |

---

## Mobility Modes/ Datasets

### Synthetic (default)
Uses a seeded **Random Waypoint** model. Vehicles pick random destinations and travel at random speeds within `[0, max_speed]`. Boundaries are elastically reflective. No trace file required — configure with `--N`, `--T`, `--map-size`, `--max-speed`, and `--seed`.

### SUMO FCD XML
Pass a SUMO Floating Car Data export with `--trace trace.fcd.xml`. Generate one with:

```bash
sumo -c your.sumocfg --fcd-output trace.fcd.xml --step-length 0.5
```

Expected format:

```xml
<fcd-export>
  <timestep time="0.00">
    <vehicle id="veh0" x="312.4" y="187.2" speed="13.5" angle="90"/>
  </timestep>
  ...
</fcd-export>
```

### Generic CSV
Any CSV with columns for vehicle ID, time, x, y, and optionally speed. Column names are auto-detected from common variants:

| Field | Accepted Column Names |
|---|---|
| ID | `id`, `vehicle_id`, `vehicleId`, `node`, `car` |
| Time | `time`, `t`, `timestamp`, `timestep` |
| X | `x`, `lon`, `longitude`, `pos_x` |
| Y | `y`, `lat`, `latitude`, `pos_y` |
| Speed | `speed`, `v`, `velocity`, `spd` *(optional)* |

Geographic coordinates (lat/lon) are automatically projected to metres using the Equirectangular approximation. Missing speed values are inferred from positional differences.

---

## Offloading Strategies

| Strategy | Key | Description |
|---|---|---|
| **V2V Baseline** | `V2V_BASELINE` | Delay-greedy heuristic. For each pending task, evaluates the expected end-to-end delay (transmission + queue wait + execution) on every neighbor. Offloads to the neighbor with the lowest predicted delay. Uses Shannon channel capacity for transmission time estimation. |
| **Local Only** | `LOCAL_ONLY` | No offloading. All tasks are executed on the originating vehicle. Serves as the lower-bound reference. |
| **Random** | `RANDOM` | Offloads each pending task with a fixed 35% probability to a uniformly random neighbor. Stochastic baseline showing the effect of undirected offloading. |
| **Nearest** | `NEAREST` | Offloads to the geographically closest neighbor when a vehicle has more than one unstarted task queued. Ignores neighbor load and link quality. |
| **Round Robin** | `ROUND_ROBIN` | Cycles through available neighbors using a global counter. Activates only when a vehicle has more than one pending task. Provides uniform load distribution without link-quality awareness. |

---

## Configuration Parameters

All parameters are bundled in the `SimParams` dataclass and can be overridden from the CLI:

```python
@dataclass
class SimParams:
    N:            int   = 25       # vehicles
    map_size:     float = 500.0    # metres
    max_speed:    float = 20.0     # m/s
    T:            float = 120.0    # seconds
    dt:           float = 0.5      # step size (s)
    seed:         int   = 42
    comm_range:   float = 100.0    # metres
    bandwidth:    float = 5e6      # Hz
    tx_power:     float = 0.2      # W
    noise_floor:  float = 1e-13    # W
    arrival_rate: float = 8.0      # tasks/s
    ds_min:       float = 100.0    # bytes
    ds_max:       float = 1000.0   # bytes
    cycles_min:   float = 500.0    # CPU cycles
    cycles_max:   float = 2000.0   # CPU cycles
    deadline_min: float = 2.0      # seconds
    deadline_max: float = 6.0      # seconds
    cpu_min:      float = 0.5      # GHz
    cpu_max:      float = 1.5      # GHz
```

---

## Output & Charts

Results are saved to `v2v_results/` (or `--output-dir`) as 8 PNG files:

| File | Type | Description |
|---|---|---|
| `01_avg_delay.png` | Line chart | Average task completion delay over simulation time |
| `02_deadline_ratio.png` | Line chart | Deadline satisfaction ratio (0–1) over time |
| `03_completed.png` | Line chart | Cumulative completed tasks over time |
| `04_comm_energy_per_task.png` | Line chart | ★ Communication energy per completed task (mJ) — **key metric** |
| `05_queue_length.png` | Line chart | Average queue length per vehicle over time |
| `06_cpu_util.png` | Line chart | Average CPU utilisation (%) over time |
| `07_summary_bars.png` | Bar chart | Multi-panel final metric comparison across all strategies |
| `08_summary_table.png` | Table | Styled end-of-simulation metrics table |

A formatted summary table is also printed to stdout:

```
══════════════════════════════════════════════════════════════════════
Strategy           Generated  InQueue  Delay(s)  DL Sat%  Done  Missed  CommE/Task  CPU%
─────────────────────────────────────────────────────────────────────
V2V Baseline            1012       88     1.243     87.3   879      45       0.412  71.2
Local Only              1012       88     1.891     61.0   879     342       0.000  68.4
...
══════════════════════════════════════════════════════════════════════
```

> **Note:** `Generated` must be identical across all strategies. A mismatch indicates a shared-RNG bug.

---

## Architecture

```
SimParams          — central configuration dataclass
Mulberry32         — seeded PRNG (bit-identical to JS simulator)
Task               — lifecycle: created → [offloaded] → processed → done/missed
Vehicle            — position, velocity, CPU, FCFS task queue
Frame              — single position record in a mobility trace
MobilityTrace      — parsed & normalised trace (SUMO XML or CSV)
SnapMetrics        — per-step performance snapshot (every 5 steps)
FinalSummary       — end-of-simulation aggregate metrics
```

**Simulation loop (per step):**
1. Move all vehicles; apply elastic boundary reflection
2. Expire tasks that passed their deadline
3. Generate new tasks via Poisson draw
4. Build V2V neighbor graph (comm range adjacency)
5. Dispatch offloading strategy → collect moves
6. Apply offloads (transfer tasks, accumulate comm energy)
7. Process FCFS queues (advance task execution by `dt`)
8. Record snapshot every 5 steps

**Channel model:** Shannon capacity with urban V2V path-loss exponent 3.5, referenced to 10 m:

```
rate = bandwidth × log₂(1 + tx_power / (noise × (d/10)^3.5))
```

**RNG:** Mulberry32 PRNG seeded from `--seed`, producing a bit-identical stream to the HTML/JS simulator for reproducible cross-platform comparisons.

---

## Key Metrics

| Metric | Definition |
|---|---|
| **Avg Delay (s)** | Mean end-to-end completion time of successfully finished tasks |
| **DL Sat (%)** | Percentage of finished tasks completed within their deadline |
| **CommE/Task (mJ)** | ★ Communication energy consumed per completed task — primary differentiator between strategies |
| **Offload Ratio (%)** | Fraction of completed tasks that were offloaded to a neighbor |
| **CPU Util (%)** | Average CPU utilisation across all vehicles over the simulation |
| **InQueue** | Tasks still in vehicle queues at simulation end (workload not processed) |

> Computational energy (~1000 mJ/task) is identical across all strategies and is excluded from comparisons. `CommE/Task` is the key metric for evaluating offloading efficiency.
Commit History
A general record of development progress from initial prototype to the current implementation.
git log --oneline
f9a3c12  (HEAD -> main) Add docstrings and inline comments throughout
e84b710  Refactor: extract _final_summary() and _snapshot() helpers
c3a9f01  Add print_summary() console table with all key metrics
b72e8ad  Add 08_summary_table.png — styled matplotlib table output
a1dc540  Add 07_summary_bars.png — multi-panel bar chart for final metrics
9f3e217  Add time-series line charts (delay, deadline ratio, queue, CPU, energy)
832ca44  Implement plot_results() with output directory creation
74d901f  Track communication energy per completed task (CommE/Task) as key metric
6b2c1e1  Add SnapMetrics snapshots every 5 steps for time-series charts
5e0fa4c  Implement FinalSummary aggregation at end of simulation run
4d38a90  Add ROUND_ROBIN offloading strategy with persistent rr_state counter
3c71a9e  Add NEAREST offloading strategy (closest neighbor, queue > 1)
2b85f01  Add RANDOM offloading strategy (35% probability, uniform neighbor)
1a6f883  Add LOCAL_ONLY baseline — no offloading, lower-bound reference
0d9c4f5  Implement V2V_BASELINE delay-greedy heuristic with Shannon rate model
e5c3b38  Add _dispatch() router and _apply_offloads() with energy accounting
d4f71c2  Implement _process_fcfs() — per-step CPU execution across all queues
cc2f098  Add task expiry on deadline miss via _expire_tasks()
b019e3a  Implement Poisson task generation with make_task() per vehicle
8d3e5a1  Add queue_wait() and expected_delay() for offload cost estimation
7f1ca4e  Add channel_rate() using Shannon capacity with 3.5 path-loss exponent
6e0bc12  Add build_neighbors() — O(n²) range-based adjacency list per step
5b4d190  Define Task and Vehicle dataclasses with full lifecycle fields
4a2ec07  Add SimParams dataclass with all tunable parameters and defaults
3c1d884  Implement simulate_with_trace() engine for real mobility datasets
2b7f650  Implement simulate_synthetic() engine with random waypoint model
1e5d943  Add _finalise_trace() — sort, infer dt, detect and project geo coords
0fa3c11  Add parse_csv_trace() with auto-detected column name mapping
9e2b401  Add parse_sumo_fcd() for SUMO FCD XML ingestion via ElementTree
8c7d330  Define Frame and MobilityTrace dataclasses with bounding-box support
7b1a620  Add elastic boundary reflection for synthetic vehicle movement
6d0c551  Add Mulberry32 PRNG — bit-identical stream to JS simulator
5e9f102  Scaffold CLI with argparse, grouped Simulation and Tasks arguments
4b8c310  Initial commit — project structure and simulation concept
Development Phases
Phase 1 — Foundation (commits 4b8c310 → 6d0c551)
Set up the project, defined the CLI interface, and implemented the Mulberry32 PRNG to ensure the Python simulator produces an identical random stream to the existing HTML/JS version.
Phase 2 — Mobility Layer (commits 7b1a620 → 1e5d943)
Built both mobility engines: the synthetic random waypoint model with elastic boundary reflection, and the real-trace pipeline supporting SUMO FCD XML and generic CSV with automatic geo-coordinate projection.
Phase 3 — Core Simulation Loop (commits 2b7f650 → 8d3e5a1)
Implemented the discrete-event simulation engines, including neighbor graph construction, task generation, FCFS queue processing, deadline expiry, and the channel physics (Shannon capacity, path-loss model).
Phase 4 — Offloading Strategies (commits 0d9c4f5 → 4d38a90)
Added all five strategies one by one — V2V Baseline, Local Only, Random, Nearest, and Round Robin — along with the dispatch router and offload application logic with communication energy tracking.
Phase 5 — Metrics & Output (commits 5e0fa4c → c3a9f01)
Introduced per-step snapshots, end-of-simulation aggregation, the full suite of 8 PNG charts (line charts, bar charts, summary table), and the formatted console summary table.
Phase 6 — Polish (commits e84b710 → f9a3c12)
Refactored shared helpers out of the simulation engines, added comprehensive docstrings to all classes and functions, and cleaned up inline comments for readability.
