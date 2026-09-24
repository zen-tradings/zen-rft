# Milestones

These milestones run in order. Each one has acceptance criteria, and it is not
done until every criterion is met and written up in
`results/milestones/M<n>.md`.

All thresholds below are initial values. Change them only by editing this file
*before* the milestone's runs, never after seeing results.

Unless a milestone says otherwise, "test" means the frozen test splits in
[`design.md`](design.md) §12.

---

## M1: `fin_numeric` data, grader, and base-model pass@k (no training)

**Scope**

- `core/task.py`, `core/numeric.py`, `core/eval.py` (val only),
  `core/manifest.py`
- `tasks/fin_numeric/`, covering:
  - companyfacts cache
  - concept map
  - statement trees
  - perturbation
  - rendering
  - templates L1–L4
  - split generation
  - grader
- Feasibility check (design §5) on Qwen3-4B, Qwen3-4B-Thinking-2507 and
  Qwen3.5-4B, served with vLLM

**Acceptance criteria**

1. **Deterministic generation.** Re-running the generator with the same seed
   and source snapshot produces byte-identical split files, with matching
   sha256 in the manifests.
2. **Target sizes built:**
   - `val` and `test` are built at their target sizes
   - `test_control` is built as unperturbed twins of test items
   - `test_postcutoff` is built at its target size
   - `train` has at least 5k items (the full 20k is not needed yet)
3. **Clean perturbation:**
   - 100% of generated items pass the exact identity checks after perturbation
   - 100% satisfy the ≥3% separation constraint
   - the item drop rate and its causes are reported per template
4. **Reference programs score 1.0.** Scoring each item's reference program
   output, formatted as an ideal answer, gives `total = 1.0` on 100% of items.
5. **Grader tests pass.** Grader unit tests cover every worked example and
   variant in `docs/tasks/fin_numeric.md` §11, plus parse edge cases:
   - no JSON
   - multiple blocks
   - string value
   - unknown unit
   - thinking-only answer
6. **Leak-free prompts.** A test asserts that no `truth` or `truth_original`
   value appears in any rendered prompt, beyond values that are legitimately
   displayed cells.
7. **Manual audit.** A reviewer reads 50 random val items (question, table and
   reference answer) and finds ≤ 1 wrong or ambiguous item. Any error found is
   fixed at the generator level, and the audit is re-run on 50 new items.
8. **Feasibility report** for each model, containing:
   - parse rate
   - pass@1, pass@4 and pass@16 per level
   - learnable fraction
   - mean length and truncation rate
   - the design §5 decision-rule outcome, written down
   - the chosen base model, with reasons (resolves Q5)
9. **Throughput re-estimate.** Measured tokens/s and GPU-hours replace the
   estimates in design §11, labelled as measured.

## M2: `fin_numeric` baselines and reporting

**Scope**

- `core/report.py`, `core/clients.py`
- Test-access guard
- FinQA/TAT-QA adapters and their audits
- Baselines: chosen base model (thinking on and off), and one or two frontier
  API models

**Acceptance criteria**

1. **Leaderboard generated.** `leaderboard.md` is generated from `results/`,
   with every column in design §12.
2. **Baseline coverage.** Every baseline has results on `test`,
   `test_control`, `test_postcutoff`, FinQA test and TAT-QA dev (arithmetic),
   each with a 95% bootstrap CI.
3. **Memorisation gap and leak rate reported** for every baseline.
4. **External-set audit.** 100 FinQA and 100 TAT-QA items are audited, and the
   exclusion list is versioned.
5. **Complete manifests.** Every run directory has a complete manifest
   (design §13), and re-scoring from `predictions.jsonl.zst` reproduces
   `metrics.json` exactly.
6. **Headroom recorded:** the gap between base and frontier on test, per
   level. If the design §6 "no headroom" condition holds, stop here and write
   a decision memo.

## M3: RFT on `fin_numeric`

**Scope**

- `core/rft.py`, `train/sft.py`
- 1–3 rounds on the chosen 4B model
- Format or supervised warm-start first, if M1's decision rule required it

**Acceptance criteria**

1. **Complete runs.** Each round's run directory has its config, manifest,
   accepted-sample statistics and sampled-completion count.
2. **Test comparison.** Test results against base, with a paired bootstrap CI.
   Checkpoints are selected on val only, and test is accessed once (checked in
   `test_access.log`).
3. **Spot check passes.** The design §9 spot check is done on the final
   checkpoint, with ≤ 2/20 flagged and no `exploit` verdict.
4. **No worse memorisation.** The memorisation gap and leak rate are no worse
   than base, beyond CI.

## M4: GRPO on `fin_numeric`, and the RL decision

**Scope**

- `train/grpo_single_turn.py` (Unsloth + TRL)
- `loss_type="dapo"` with the design §7.2 settings
- Ablations:
  - `dr_grpo` with `scale_rewards="none"`
  - smooth vs binary numeric reward
- GRPO matched to RFT on sampled-completion budget
- 3 seeds for the headline comparison

**Acceptance criteria**

1. **Headline comparison.** GRPO vs RFT vs base on test, FinQA and TAT-QA,
   with paired CIs over 3 seeds, at matched budget.
2. **Training curves logged** (design §12). Include the zero-variance-group
   fraction and the length trend.
3. **pass@k analysis.** pass@k curves up to k = 64 for base, RFT and GRPO on
   the fixed 500-item subset.
4. **Spot check passes,** with detector rates reported.
5. **Written decision memo.** It says whether RL is worth it for
   `fin_numeric` under the design §6 criteria (a)–(d), and which model goes
   forward (RFT or GRPO).
6. **Optional:** a DPO run with the same reporting, if time allows.

## M5: `fin_tools` environment, grader, and base-model pass@k (no training)

**Scope**

- `tasks/fin_tools/`:
  - logical schema and sources
  - messiness transforms X0–X3
  - DB build
  - the five tools with limits
  - fault injection
  - question templates
  - reference solver
  - grader
- verifiers adapter
- **Framework spike:** one masked, multi-turn LoRA GRPO step with prime-rl on
  one GPU

**Acceptance criteria**

1. **Deterministic DB builds.** Rebuilding an instance from its seed and
   config gives an identical file hash. At messiness levels X2–X3, instances have about 250–400 tables,
   or the target is revised with reasons.
2. **Reference solver replays.** 100% of generated items are solved by the
   scripted reference solver through the real tools, within limits, and
   reproduce the `truth.db` answer.
3. **Tool tests pass:**
   - every error type in `docs/tasks/fin_tools.md` §4 is triggered and checked
   - read-only guards resist a battery of write, `ATTACH` and `PRAGMA`
     attempts
   - the time limit fires
   - output caps truncate at row boundaries
4. **Grader tests pass,** covering every worked example and variant in the
   spec §12, plus grounding-gate, fault-exemption and penalty-cap cases.
5. **Mask correctness test.** One rendered trajectory yields a loss mask over
   exactly the assistant tokens, in the chosen framework. Covers the
   verifiers/prime-rl spike, or the TRL fallback if the spike fails.
6. **Split separation.** `test_ood_schema` physical table names are disjoint
   from every training instance's, checked by a test. `test_ood_domain` table
   families are absent from training instances.
7. **Manual audit.** A reviewer reads 30 random val questions with their
   reference trajectories and finds ≤ 1 wrong or ambiguous item.
8. **Feasibility report:**
   - base-model pass@k per level, at messiness X1 and X3, at fault rates 0 and 0.1
   - a frontier baseline through the same verifiers environment
   - the design §5 decision-rule outcome

## M6: `fin_tools` training and generalisation

**Scope**

- RFT on trajectories
- GRPO via prime-rl, or the fallback chosen in M5
- Curriculum per `docs/tasks/fin_tools.md` §10

**Acceptance criteria**

1. **Results on every split:** base, frontier, RFT and GRPO on `test_iid`,
   `test_ood_schema` and `test_ood_domain`, at fault rates {0, 0.1, 0.3}, with
   paired CIs.
2. **Robustness and recovery reported:** robustness and
   `fault_recovery_rate` for each method.
3. **Spot checks on full trajectories** pass.
4. **Decision memo** using the same design §6 criteria. Criterion (b) uses
   `test_ood_schema` as the OOD split.

## M7: Integration pilots

**Scope**

- Serve the best adapter(s) with vLLM multi-LoRA
- Grader-derived routing labels for coding-routing-benchmark
- zen-fundamentals extractor pilot proposal

**Acceptance criteria**

1. **vLLM serving matches eval.** One vLLM instance serves both task adapters,
   and its measured p50/p95 latency and $/1k items match the leaderboard
   figures within 20%.
2. **Routing labels produced.** A `fin_dev_v1.jsonl` prompt file is produced
   in coding-routing-benchmark's schema, with outcome-derived
   `reference_model` labels. The upstream changes it needs are filed as an
   issue or PR there.
3. **Extractor pilot plan.** A written plan for a zen-fundamentals `extractor`
   pilot, with the replay eval that would decide it. The plan only, no
   integration code.
