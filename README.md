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

## Mobility Modes

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
