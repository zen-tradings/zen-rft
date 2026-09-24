# zen-rft

zen-rft trains small open-weight models, starting with Qwen3-4B plus LoRA, to
get better at specific financial tasks, using graders that can be checked in
code. Each task has one grader that serves as the reward during training, the
metric during evaluation, and the evidence for deciding when a cheap trained
model can stand in for a frontier model. Training uses existing trainers
(Unsloth/TRL, and verifiers with prime-rl). What this repo adds is the tasks
and graders: `fin_numeric` (numerical reasoning over perturbed 10-K/10-Q
excerpts, with ground truth from SEC XBRL) and `fin_tools` (multi-turn tool
use over a deliberately messy SQLite database of funds, prices and
fundamentals).

Part of [zen-tradings](https://github.com/zen-tradings), alongside zen-coding,
zen-fundamentals and coding-routing-benchmark.

## Status

**Design only.** There is no code, data or results yet. Targets and estimates
in the docs are labelled as such. The next step is Milestone 1: `fin_numeric`
data generation, the grader, and base-model pass@k, with no training.

## Repo layout (planned)

```
zen-rft/
├── core/        # Task/Environment interfaces, shared numeric grading, eval, reporting, rejection-sampling loop
├── train/       # entry points: SFT, GRPO single-turn (Unsloth + TRL), multi-turn (verifiers + prime-rl), DPO
├── tasks/
│   ├── fin_numeric/   # XBRL question generation, perturbation, rendering, grader
│   └── fin_tools/     # SQLite environment, tools, fault injection, grader
├── results/     # per-run configs, manifests, metrics, spot checks; generated leaderboards
├── data/        # gitignored: source caches, generated splits, DB instances
└── docs/
```

## Docs

- [Design](docs/design.md): interfaces, methods, frameworks, compute,
  evaluation, integration
- [Task spec: fin_numeric](docs/tasks/fin_numeric.md)
- [Task spec: fin_tools](docs/tasks/fin_tools.md)
- [Milestones](docs/milestones.md)
- [Open questions](docs/open_questions.md)
