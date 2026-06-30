# WorkBenchMark

A LEGO Duplo assembly benchmark for robotic manipulation research, introduced in the paper:

> **WorkBenchMark: A LEGO-Based Assembly Benchmark with an Assembly-by-Disassembly Baseline for the Smart Manufacturing League**

> arXiv:2606.19358 — [https://arxiv.org/abs/2606.19358](https://arxiv.org/abs/2606.19358)

WorkBenchMark provides 400 procedurally generated assembly tasks across four difficulty tiers, designed to evaluate robotic systems that must couple low-level manipulation, active perception, and task-level symbolic reasoning. It was inspired by the RoboCup Smart Manufacturing League.

## Repository Structure

```
benchmark_tasks/          # what solvers see — target assemblies only
  tier1/                  # 100 tasks — 2-block assemblies
  tier2/                  # 100 tasks — 3-block assemblies
  tier3/                  # 100 tasks — 4–7 block assemblies
  tier4/                  # 100 tasks — 6–12+ block assemblies

ground_truth/             # for evaluation only — do not use to solve tasks
  tier1/
  tier2/
  tier3/
  tier4/
```

Each task is a YAML file (`task_001.yaml` … `task_100.yaml`). Files in `benchmark_tasks/` contain only the target assembly (`blocks:`). Files in `ground_truth/` contain the same target plus the hidden initial layout of bricks on the setup plate (`initial_blocks:`), used to evaluate perception accuracy during scoring.

> **Note:** `ground_truth/` must not be read by solver code. The benchmark is perception-based, i.e., the robot must locate bricks on the setup plate using its own sensors, not from the file.

## Task Format

### `benchmark_tasks/` — solver input

Each file contains a single `blocks` key listing every brick in the target structure:

```yaml
blocks:
- name: 4x2_brick_1       # unique identifier within the task
  type: brick_4x2         # brick_4x2 or brick_2x2
  color: green            # red | blue | green | yellow
  pos:                    # [x, y, z] in metres; y is vertical
  - 0.0
  - 0.148
  - 0.0
  rotation:               # [roll, pitch, yaw] in degrees
  - 0
  - 0
  - 0
```

### `ground_truth/` — evaluator input

The same `blocks` section, plus `initial_blocks` describing where each brick sits on the setup plate before manipulation. Brick `name` values match across both sections, linking each physical brick to its role in the target assembly.

```yaml
blocks:
  # ... same as benchmark_tasks file ...

initial_blocks:
- name: 4x2_brick_1
  type: brick_4x2
  color: green
  pos:                    # scattered position on the setup plate
  - 0.3776
  - -0.114              # negative y = setup plate (below assembly area)
  - 0.0495              # all initial bricks lie flat at this height
  rotation:
  - 0
  - 0
  - 0
```

### Fields

| Field | Description |
|---|---|
| `name` | Unique label for the brick within this task. Shared between `blocks` and `initial_blocks`. |
| `type` | Brick geometry: `brick_4x2` (4×2 studs) or `brick_2x2` (2×2 studs). |
| `color` | One of `red`, `blue`, `green`, `yellow`. |
| `pos` | 3D position `[x, y, z]` in metres. The `y` axis is vertical. Assembly positions have positive y; setup plate positions have negative y. |
| `rotation` | Orientation as `[roll, pitch, yaw]` in degrees. All initial bricks are flat (rotation 0). |


## Tiers

Tiers define assembly complexity, primarily by the number of bricks and the spatial intricacy of the target structure:

| Tier | Tasks | Blocks per task | Complexity |
|---|---|---|---|
| **Tier 1** | 100 | 2 | Minimal — two bricks stacked or placed side-by-side. Entry-level pick-and-place. |
| **Tier 2** | 100 | 3 | Simple — three bricks requiring correct sequencing and basic spatial reasoning. |
| **Tier 3** | 100 | 4–7 | Intermediate — multi-layer structures with rotation variation; partial occlusion. |
| **Tier 4** | 100 | 6–12+ | Complex — large multi-layer assemblies requiring extended planning horizons. |

All tasks share the same two brick types (`brick_4x2`, `brick_2x2`) and four colors (`red`, `blue`, `green`, `yellow`), so the difficulty gradient is driven entirely by structural complexity rather than parts diversity.


## Using the Benchmark

### Loading a task (solver)

```python
import yaml

with open("benchmark_tasks/tier1/task_001.yaml") as f:
    task = yaml.safe_load(f)

for brick in task["blocks"]:
    print(brick["name"], brick["type"], brick["color"], brick["pos"])
```

### Loading ground truth (evaluator)

```python
import yaml

with open("ground_truth/tier1/task_001.yaml") as f:
    gt = yaml.safe_load(f)

target   = {b["name"]: b for b in gt["blocks"]}
initial  = {b["name"]: b for b in gt["initial_blocks"]}
```

### Evaluation approach

The paper proposes an **assembly-by-disassembly** baseline: the robot first observes a completed reference structure and then reasons about the reverse sequence of moves needed to build it. A solution is scored by how many bricks are placed correctly (matching type, color, position, and orientation within a tolerance).

Typical metrics used in the benchmark:

- **Block accuracy** — fraction of bricks placed at the correct position and orientation.
- **Task success rate** — fraction of tasks where all bricks are placed correctly.
- **Completion rate by tier** — success rate broken down per tier, showing scaling behaviour.
- **Perception accuracy** — how closely the robot's perceived initial brick positions match the ground truth `initial_blocks` positions (evaluated separately from assembly quality).

### Running all tiers

```bash
# Solver: iterate over every task across all tiers
for tier in 1 2 3 4; do
  for task in benchmark_tasks/tier${tier}/task_*.yaml; do
    python solve.py --task "$task"
  done
done

# Evaluator: score results against ground truth
for tier in 1 2 3 4; do
  for gt in ground_truth/tier${tier}/task_*.yaml; do
    python evaluate.py --ground-truth "$gt"
  done
done
```

## Citation

```bibtex
@inproceedings{ma2026workbenchmark,
  title         = {WorkBenchMark: A LEGO-Based Assembly Benchmark with an
                   Assembly-by-Disassembly Baseline for the Smart Manufacturing League},
  author        = {Ma, Wenbo and Swoboda, Daniel and Tschesche, Matteo and Hofmann, Till},
  booktitle     = {RoboCup 2026: Robot World Cup XXIX},
  series        = {Lecture Notes in Computer Science},
  publisher     = {Springer},
  address       = {Incheon, South Korea},
  year          = {2026},
  eprint        = {2606.19358},
  archivePrefix = {arXiv},
  note          = {Oral presentation, RoboCup Symposium 2026}
}
```
