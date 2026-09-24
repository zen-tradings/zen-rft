# zen-rft design

Status: **design only**. No code, data or results exist yet.

- Every number marked *target*, *estimate* or *initial value* is a guess.
- External facts (framework features, prices, model releases) were checked
  on 2026-09-24. Sources are listed in §16.

Task specs:

- [`tasks/fin_numeric.md`](tasks/fin_numeric.md)
- [`tasks/fin_tools.md`](tasks/fin_tools.md)

Contents:

1. [Purpose](#1-purpose)
2. [Core idea: one grader, three uses](#2-core-idea-one-grader-three-uses)
3. [The two tasks](#3-the-two-tasks)
4. [Shared interfaces](#4-shared-interfaces)
5. [Feasibility check before training](#5-feasibility-check-before-training)
6. [Methods and hypotheses](#6-methods-and-hypotheses)
7. [Training algorithms](#7-training-algorithms)
8. [Frameworks](#8-frameworks)
9. [Reward hacking and spot checks](#9-reward-hacking-and-spot-checks)
10. [Base models](#10-base-models)
11. [Compute plan](#11-compute-plan)
12. [Evaluation protocol](#12-evaluation-protocol)
13. [Reproducibility](#13-reproducibility)
14. [Repo layout](#14-repo-layout)
15. [Integration with other zen-tradings repos](#15-integration-with-other-zen-tradings-repos)
16. [External facts and sources](#16-external-facts-and-sources)

---

## 1. Purpose

zen-rft trains small open-weight models (about 4B–9B parameters, LoRA) to do
specific financial tasks better, using graders that can be checked in code.

The org builds auditable quantamental research agents. This repo adds the
answer to one question those agents keep running into: *for this narrow task,
can a small trained model replace an expensive frontier call, and how do we
know?*

**The value here is the tasks and graders, not a training framework.** All
training uses existing trainers.

## 2. Core idea: one grader, three uses

Each task has exactly one grader: a pure, deterministic function
`score(item, parsed) → {total, components}`. The same function, at the same
version, serves three purposes:

| Use | How the grader is used |
|---|---|
| **Training reward** | `total` is the scalar reward for RFT filtering, GRPO advantages, and DPO pair construction |
| **Evaluation metric** | mean `total` plus task headline metrics (`acc@0.5%`, F1) on frozen test splits; `components` explain failures |
| **Routing evidence** | per-item scores across candidate models, including frontier APIs, answer "which is the cheapest model that is good enough for this item class" (§15) |

Consequences:

- **No grader change without a version bump.** No results-table column mixes
  grader versions. Each column states the grader it uses, e.g. FinQA/TAT-QA
  columns use the adapter `fin_numeric/ext@x`.
- **Graders import nothing from training frameworks.** Routing and eval code
  must be able to use them without GPU dependencies.
- **A grader bug is a reward bug.** Graders get the most unit tests in the repo,
  and every worked example in the task specs becomes a test case.

*Alternative:* separate training rewards (shaped, dense) from evaluation
metrics (strict). That is common practice and can speed up learning. But it
lets training optimise something other than what we report, and the routing
evidence would then describe a different target. We allow only tightly
bounded exceptions, such as the optional early-curriculum shaping in
`fin_tools` §7.4, and never in headline runs.

## 3. The two tasks

| | `fin_numeric` | `fin_tools` |
|---|---|---|
| Turns | single | multi-turn with tools |
| Input | rendered 10-K/10-Q tables and text, perturbed | natural-language question plus a messy SQLite DB of 250–400 tables (target) |
| Output | reasoning plus JSON `{value, unit, scale, formula, cited_rows}` | tool calls, then a JSON answer (scalar, list, or entity-value pairs) |
| Ground truth | reference program over displayed values; source is SEC XBRL company facts | reference SQL on a hidden clean DB, replayed through the real tools |
| Grader | smooth relative-error decay × sign and scale gates, plus citations | correctness (numeric or F1) × (1 − bounded penalties) |
| Contamination control | value perturbation with a separation constraint; unperturbed control split | perturbed fundamentals, rescaled prices, synthetic fund books |
| Generalisation test | CIK-disjoint splits, paraphrase hold-out, FinQA / TAT-QA | unseen questions, **unseen tables**, unseen domains, fault rates |
| Target sizes | train 20k / val 1k / test 2k / control 1k | train 6k / val 500 / test_iid 1k / ood_schema 500 / ood_domain 300 |

Full specifications, grader definitions and worked examples are in the task
docs.

## 4. Shared interfaces

### 4.1 Does verifiers already provide the Task abstraction?

**Largely yes.** Prime Intellect's `verifiers` (MIT) has two APIs:

- **v1** (`verifiers.v1`): a `Taskset` with a `load()` method, `vf.Task`
  classes, and scoring written as `@vf.reward` / `@vf.metric` functions over
  a `vf.Trace`. It also has Toolsets and runtimes.
- **Legacy v0**: `SingleTurnEnv`, `MultiTurnEnv`, `ToolEnv` and
  `StatefulToolEnv`, together with `Rubric` and `Parser`. It is deprecated and
  "will be fully removed in a future release".

That covers dataset loading, prompting, parsing, scoring, multi-turn rollout,
and evaluation against OpenAI-compatible endpoints.

**Decision: build on verifiers for everything it covers**:

- multi-turn rollout and environment execution
- evaluation runs against local and API models
- packaging tasks as installable environments

We do **not** write our own rollout loop or environment runtime.

We keep a thin, framework-free `core/task.py` contract (below), for three
reasons:

1. The single-turn trainer (TRL) and the routing benchmark need the grader
   without verifiers.
2. verifiers is migrating from v0 to v1, and that churn should stay behind one
   adapter file per task (`tasks/<task>/vf_env.py`).
3. If the multi-turn trainer choice changes (§8), only the adapter changes.

The contract is deliberately *narrower* than verifiers. It covers what a
grader and a tool implementation need, and nothing about rollout scheduling.

*Alternative:* implement tasks directly as verifiers v1 Tasksets with no
contract of our own. That means less code. But our graders would then depend
on a pre-1.0-stable API, and the TRL path and routing benchmark would have to
depend on verifiers too.

### 4.2 Task interface (every task)

```python
# core/task.py
from dataclasses import dataclass, field
from typing import Any, Mapping, Protocol, Sequence

Message = dict[str, Any]          # OpenAI-style chat message


@dataclass(frozen=True)
class Item:
    """One frozen evaluation/training example."""
    id: str
    split: str
    payload: Mapping[str, Any]    # model-visible inputs (tables, text, question, db instance id)
    truth: Mapping[str, Any]      # hidden ground truth; never rendered into prompts
    meta: Mapping[str, Any]       # level, template id, traps, generator git sha, seed


@dataclass(frozen=True)
class Score:
    """Grader output. `total` is the reward and the metric."""
    total: float                              # in [0, 1]
    components: Mapping[str, float]           # per-component breakdown, e.g. numeric, citation, sign_ok
    info: Mapping[str, Any] = field(default_factory=dict)  # non-numeric diagnostics, e.g. rel_error, leak_hit


class Task(Protocol):
    name: str                 # "fin_numeric"
    grader_version: str       # "fin_numeric/grader@1.0.0"

    def load(self, split: str) -> Sequence[Item]:
        """Return the frozen items of `split`.

        Verifies the split file's sha256 against its manifest and raises on
        mismatch. Loading any test split is recorded in
        `results/test_access.log` (see §12).
        """

    def to_prompt(self, item: Item) -> list[Message]:
        """Render model-visible messages for `item`.

        Must be deterministic and must not read `item.truth`. A unit test
        asserts that no truth value appears in the rendered prompt.
        """

    def parse(self, output: Any) -> Any | None:
        """Turn raw model output into a typed answer, or None if unparseable.

        Single-turn tasks receive the final assistant text; multi-turn tasks
        receive a `Trajectory`. Pure: no I/O, no randomness.
        """

    def score(self, item: Item, parsed: Any | None) -> Score:
        """Grade a parsed answer against `item.truth`.

        Pure and deterministic. `parsed is None` must give total == 0.
        Every component used in `total` must appear in `components`.
        """
```

### 4.3 Environment interface (multi-turn tasks)

```python
# core/task.py (continued)
from typing import Literal


@dataclass(frozen=True)
class ToolEvent:
    """One executed tool call, as the grader sees it."""
    index: int
    name: str
    arguments: str                    # raw JSON string exactly as generated
    ok: bool
    kind: Literal["ok", "invalid", "model_error", "injected"]
                                      # invalid → p_invalid; model_error → may count as unrecovered;
                                      # injected → exempt from penalties
    error_type: str | None            # "InvalidToolCall", "SQLError", "InvalidArgument", "Timeout", ...
    tables: tuple[str, ...]           # tables/entities referenced (used for the "recovered" rule)
    elapsed_ms: float


@dataclass
class StepResult:
    messages: list[Message]           # tool-role messages to append to the conversation
    done: bool                        # step limit reached
    info: Mapping[str, Any]


@dataclass
class Trajectory:
    messages: list[Message]           # full conversation, including tool results
    tool_events: list[ToolEvent]
    final_output: str | None          # last assistant message with no tool call, if any
    stop_reason: Literal["final_answer", "step_limit", "context_limit", "error"]


class Environment(Protocol):
    def tool_schemas(self) -> list[dict]:
        """OpenAI function-calling schemas for the tools this env exposes."""

    def reset(self, item: Item, seed: int) -> list[Message]:
        """Start an episode: open a fresh read-only connection to the item's DB
        instance, set up fault draws u_k = hash(item.id, seed, k) where
        `seed` is the GRPO group seed (shared by the group's rollouts), and
        return the initial messages (system prompt + question)."""

    def step(self, assistant_message: Message) -> StepResult:
        """Execute the tool calls in `assistant_message` (in order, with limits
        and fault injection) and return the resulting tool messages. Invalid
        calls return an error message rather than raising."""

    def trajectory(self) -> Trajectory:
        """The episode so far, for parsing and scoring."""

    def close(self) -> None:
        """Release the DB connection and temp resources."""


class MultiTurnTask(Task, Protocol):
    max_tool_calls: int
    max_turns: int

    def make_env(self, config: Mapping[str, Any]) -> Environment:
        """Build an environment. `config` holds fault rates, limits and the
        messiness level. The framework adapter (verifiers) drives this object;
        our own code never implements a rollout loop."""
```

`tasks/fin_tools/env.py` implements `Environment` in plain Python: the DB,
the tools, limits and faults. `tasks/fin_tools/vf_env.py` wraps it as a
verifiers tool environment. The per-rollout DB handle and fault RNG are
hidden, injected state: v0 `StatefulToolEnv` `args_to_skip`, or the v1
equivalent, to be confirmed in the Milestone 5 spike.

## 5. Feasibility check before training

RL only learns from prompts where the model *sometimes* succeeds. So before
any training, per task and per candidate base model:

1. Take 200 `val` items, stratified by level. This is a *target*.
2. Sample **k = 16** outputs per item using the model card's recommended
   sampling settings. Take 8 if the budget is tight.
3. Define *pass* as follows:
   - `fin_numeric`: `rel_error ≤ 0.5%` with the sign and scale gates passed
   - `fin_tools`: `C ≥ 0.9`
4. Report per level:
   - parse rate
   - pass@1, pass@4 and pass@16, using the unbiased estimator
     `1 − C(n−c, k)/C(n, k)` (Chen et al., 2021)
   - **learnable fraction**: items with 1 ≤ passes ≤ k−1, which are exactly the
     items that give GRPO a non-zero advantage
   - mean output length and truncation rate

**Decision rule.** All thresholds are *initial guesses*, and the outcome is
recorded in the milestone report.

| Condition (per task, per level) | Action |
|---|---|
| parse rate < 80% | **format warm-start** first: SFT on ≤2k correctly formatted outputs (filtered base or frontier samples), then re-check |
| pass@16 < 15% | **supervised warm-start**: SFT on grader-verified traces (RFT from easier levels, or frontier traces that score ≥ 0.9), or drop the level from the RL curriculum |
| learnable fraction < 25% | fix the curriculum before RL: re-weight levels toward the ones where the model is between "always" and "never" |
| pass@1 > 85% | the level is saturated: keep it in eval, exclude it from RL sampling |
| otherwise | proceed to RFT and GRPO from the base model |

*Alternative:* skip the check and always SFT-warm-start. That is safer, but
it costs frontier API spend on traces and hides whether RL alone would have
worked. The check itself costs about one GPU-hour (§11).

## 6. Methods and hypotheses

All methods use the same graders, splits and decoding settings. They are run
in this order:

| # | Method | Hypothesis | Result that would count against it |
|---|---|---|---|
| 1 | **Baselines**: base model (thinking on and off), and one or two frontier API models | A clear gap exists between base 4B and frontier on L3/L4 and D3/D4, which is the headroom for training | Base 4B within about 3 points of frontier on test: training can only win on cost, and routing should just use the base model |
| 2 | **Rejection-sampling fine-tuning (RFT)**: sample, keep outputs with `total ≥ 0.9`, SFT, iterate 1–3 rounds | Most of the achievable gain comes from filtering the model's own successes, and it is cheap and stable | No significant gain over base after 2 rounds: the model rarely finds correct solutions, so revisit warm-start and difficulty before trying RL |
| 3 | **GRPO** with a length-bias-corrected loss (§7) | At a **matched sampled-completion budget**, GRPO beats RFT, especially on L3/L4, the OOD splits (FinQA/TAT-QA, `test_ood_schema`) and fault robustness | See "RL is not worth it" below |
| 4 | **DPO** on grader-constructed pairs (optional) | Offline pairs recover most of the GRPO gain at lower cost | DPO < RFT: drop it |

**When RL is not worth it.** Any one of these means we ship RFT and do not
invest further in RL for that task:

- (a) GRPO − RFT < 2 points of mean test score, with a paired-bootstrap 95%
  CI that includes 0, at matched budget.
- (b) GRPO's gain is only in-distribution: no gain, or a loss, on FinQA/TAT-QA
  or `test_ood_schema`.
- (c) GRPO's gain comes with a larger memorisation gap, a higher `leak_hit`
  rate, or a spot-check flag rate above the §9 threshold. That points to reward
  exploitation, not skill.
- (d) GRPO costs more than 3× RFT GPU-hours for less than 3 points.

All thresholds are initial and are fixed before the runs.

**Also reported: pass@k curves up to k = 64** on a 500-item subset, for base,
RFT and GRPO. Earlier work reports that RLVR tends to raise pass@1 while
narrowing pass@k at large k (Yue et al., 2025). We report whether training
*sharpens* what the base can already do or *extends* it. Either outcome is
informative, and neither decides the comparison on its own.

**Routing-relevant target (a target, not a result).** A trained 4B reaches
≥ 90% of the best frontier model's test score at ≤ 1/10 of its cost per item.

## 7. Training algorithms

### 7.1 RFT (`core/rft.py`)

Each round:

1. Sample k = 8 completions for 5k train prompts (a *target*).
2. Keep completions with `total ≥ 0.9`, at most 2 per prompt, deduplicated.
3. SFT a fresh LoRA **from the base model** on the union of all accepted
   samples so far.
4. Sample the next round from the new model.

This is the STaR / ReST / expert-iteration pattern.

*Alternative:* continue from the previous round's adapter. That is faster,
but errors compound across rounds and the result depends on the path taken.

**Budget accounting.** The loop is shared by both tasks. For `fin_tools`, a
"completion" is a full trajectory, and the SFT loss uses the same assistant-only
mask as RL. Its sampled-completion count is the budget GRPO is matched
against.

### 7.2 GRPO: length-bias-corrected loss

Outputs are long whenever thinking mode is on. That makes length bias a real
concern:

- **Standard GRPO** divides each sequence's loss by its own length. This
  under-penalises long wrong answers, and reported training runs show wrong
  answers growing longer.
- **Dr. GRPO** (Liu et al., 2025) removes both the per-response length
  normalisation and the std normalisation of advantages.
- **DAPO** (Yu et al., 2025) uses:
  - token-level policy-gradient loss
  - Clip-Higher (decoupled clip ranges)
  - dynamic sampling (drop prompts whose samples all succeed or all fail)
  - overlong filtering and shaping

**Initial configuration for single-turn**, on TRL `GRPOTrainer`:

| Setting | Value | Note |
|---|---|---|
| `loss_type` | `"dapo"` | TRL's current default; token-level normalisation |
| `mask_truncated_completions` | `True` | DAPO overlong filtering: truncated outputs do not push the policy |
| `scale_rewards` | `"group"`; ablation `"none"` | `"none"` is the Dr. GRPO setting |
| `num_generations` | 8–16 | 16 when the learnable fraction is low |
| KL coefficient `beta` | 0.0; ablation 0.01 | DAPO drops the KL term |
| Dynamic sampling | offline: every N steps, re-score the pool and drop all-pass/all-fail prompts | approximates DAPO's online filter without framework changes |
| LoRA | rank 32, all linear layers; ablation rank 8 / 64 | see §10 |
| Rollout engine | vLLM colocated with the trainer | single GPU |

- **Ablation:** `loss_type="dr_grpo"` with `scale_rewards="none"`.
- **Multi-turn** uses the same loss family in prime-rl. The exact knobs are
  mapped in the Milestone 5 spike.

*Alternative:* PPO with a value model. It gives better per-token credit
assignment, but needs a second network and more memory, and on outcome-reward
tasks GRPO-family methods are reported to reach similar results.

### 7.3 DPO (optional)

- **Pairs:** same prompt; chosen has `total ≥ 0.9`; rejected has
  `total ≤ 0.3`. Prefer rejected samples that parse (informative errors) over
  format failures.
- **Scope:** `fin_numeric` only at first. Multi-turn DPO over trajectories is
  less standard, and masking is harder to get right.

## 8. Frameworks

### 8.1 Single-turn (`fin_numeric`): Unsloth + TRL

- TRL `GRPOTrainer` supports the loss variants in §7.2, plus
  `mask_truncated_completions`, `scale_rewards` and colocated vLLM.
- Unsloth publishes a Qwen3-4B GRPO notebook and claims large VRAM savings.
  Those savings are what make one GPU enough.
- *Alternative:* run `fin_numeric` through verifiers/prime-rl as well. That
  means one stack for both tasks, but loses Unsloth's single-GPU memory
  savings. We revisit this if maintaining two stacks turns out costly.

### 8.2 Multi-turn (`fin_tools`): comparison

| | **verifiers + prime-rl** | **OpenPipe ART** | **verl (agent loop)** | **TRL `environment_factory`** |
|---|---|---|---|---|
| Env / task abstraction | yes: tool environments (v0) and Tasksets/Toolsets (v1), rubrics, parsers | no env abstraction: you write a rollout function that returns `Trajectory` objects | `AgentLoopBase` plus `BaseTool` (YAML config or `@function_tool`) | env class whose methods are tools (experimental) |
| Tool tokens masked from loss | yes (prime-rl: one sample, one loss mask, token-in) | not found in docs (unverified) | yes (`response_mask`: 0 for tool tokens) | yes (`tool_mask`) |
| LoRA | yes | yes (LoRA only) | yes (PEFT on FSDP) | yes (PEFT) |
| Single GPU | yes (supports single-GPU debugging) | yes (`LocalBackend` pauses inference while training) | runs on 1 GPU (quickstart), built for clusters | yes |
| Eval against API models with the same env | yes | via your rollout code | not its focus | no |
| Maturity / churn | active; **v0 → v1 migration underway** | active (v0.5.19, Aug 2026) | mature, heavy config; repo moved to `verl-project/verl` | experimental; needs transformers ≥ 5.2 |
| Fit with the single-turn stack | separate stack | shares Unsloth | separate stack | same stack as `fin_numeric` |
| License | MIT / Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |

**Pick: verifiers (environment) + prime-rl (trainer).** Reasons:

1. **It already is the Task/Environment abstraction the brief asks for.**
   Using it means we don't duplicate one (§4.1).
2. Tool-token masking is documented. So is token-in/token-out training, which
   matters because Qwen3 templates re-render earlier thinking differently
   (`fin_tools` §10).
3. The same environment runs baseline evaluations against frontier APIs and
   the local model. That keeps method 1 and methods 2–4 identical.
4. LoRA and single-GPU runs are supported.

Risks, and how we handle them:

- **v0/v1 churn:** pin a version, and keep all verifiers code in one adapter
  file per task.
- **Unknowns:** the Milestone 5 spike (`milestones.md`) must show a masked
  multi-turn LoRA step on one GPU before we commit.
- **Fallback:** TRL `environment_factory`. It keeps one stack across both
  tasks, and we accept that it is experimental.
- **Why not the others:** ART has no environment abstraction and its masking
  is unverified. verl is the heaviest option for a single-GPU LoRA setting.

## 9. Reward hacking and spot checks

The per-grader exploit tables are in the task specs (`fin_numeric` §9,
`fin_tools` §11). Summary of the most likely exploits:

| Grader | Most likely exploits | Main guards |
|---|---|---|
| `fin_numeric` | recalling public figures; citation spam; unit juggling; right number from wrong reasoning; template-wording overfit | perturbation with ≥3% separation plus a `leak_hit` metric; F1 citations with an existence gate; fixed unit families; `formula_consistent` diagnostic; paraphrase hold-out plus FinQA/TAT-QA |
| `fin_tools` | guessing entities; list spam; table dumps; DB tampering; reading hidden truth; burning retries | grounding gate; F1; row, output and time caps plus a budget penalty; read-only ×3 plus an authorizer; sandboxed paths; budget grows exactly one per injected fault |

**Automated detectors**, run on every eval and on a sample of training
rollouts. Each is logged as a rate per checkpoint:

- `leak_hit`: the answer matches the unperturbed truth
- `formula_consistent = 0` while `numeric = 1`
- `grounding_dropped > 0`
- output-length outliers (> p99 of base)
- repeated identical tool calls
- more than one JSON block
- reward rising while `acc@0.5%` or F1 stays flat, which suggests the model is
  exploiting smooth partial credit

**Manual spot-check procedure**, at every evaluated checkpoint and before
any result goes into a results table:

1. **Sample** 20 outputs with `total ≥ 0.9`, stratified by level. Add up to 10
   more that set off any detector above, and 5 more whose score rose most
   against the previous checkpoint.
2. **Review.** A reviewer reads each output with the task-specific checklist
   (`fin_numeric` §9, `fin_tools` §11). The verdict is `ok` / `suspicious` /
   `exploit`, with a one-line note.
3. **Record** in `results/<run>/spotcheck.md`: item IDs, verdicts, reviewer,
   date.
4. **Threshold.** If more than 2 of the 20 random high scorers are
   `suspicious` or `exploit`, or there is any `exploit` at all, training on
   that task stops. Then fix the grader or data, bump the grader version, and
   re-score the affected results. Results under the old grader version are
   marked superseded, never deleted.
5. **Blinding.** Where possible the reviewer does not know which method
   produced the output. Method names are hidden in the review file.

## 10. Base models

**Start: Qwen3-4B, then Qwen3-8B, both with LoRA.**

Why Qwen3-4B:

- Apache-2.0.
- 32,768-token native context (131k with YaRN), enough for filing excerpts
  and 24k-token tool trajectories.
- Thinking and non-thinking modes in one checkpoint. That lets us measure
  whether long reasoning is worth its cost on these tasks without changing
  models.
- A native tool-calling chat template.
- First-class support in every tool we plan to use: Unsloth (with a GRPO
  notebook), TRL, vLLM, prime-rl and verl.
- A 4B LoRA RL run fits on one 80 GB GPU with room for colocated vLLM.

Why 8B second: same family and tokenizer. That isolates the effect of scale,
which tells routing whether the extra serving cost of 8B buys accuracy.

**Newer candidates exist.** The brief predates them, so they are flagged here:

- **Qwen3-4B-Instruct-2507 / Thinking-2507** (Aug 2025, 262k context). These
  split the modes into separate checkpoints.
- **Qwen3.5-4B / 9B** (Feb 2026, Apache-2.0, vision-language, 262k native
  context). They are likely stronger, but LoRA-RL support for the Qwen3.5
  architecture in Unsloth, TRL and prime-rl has not been verified by us.

**Decision:** run the §5 feasibility check on Qwen3-4B, Qwen3-4B-Thinking-2507
and Qwen3.5-4B, which is about 1 GPU-hour each. Pick the one with the best
learnable fraction *among those our training stack supports end to end*. The
scale step follows the chosen family (8B, or Qwen3.5-9B). This is Q5 in the
open questions.

**Why LoRA rather than full fine-tuning:**

- one GPU is enough
- adapters are small, so every run's weights can be kept
- one base model can serve many task adapters (§15)
- Thinking Machines' "LoRA Without Regret" (2025) reports LoRA matching full
  fine-tuning for RL even at low ranks, because RL delivers few bits per
  episode

*Alternative:* full fine-tuning of the 4B. It has a higher ceiling on SFT-heavy
data, but needs multi-GPU memory, and serving a full model per task removes the
multi-LoRA advantage.

## 11. Compute plan

**All figures are estimates.**

- **Prices (checked 2026-09-24, on-demand per GPU-hour):**
  - H100 80 GB: RunPod $2.69–3.49, Modal $3.95, Lambda $3.29–4.29
  - A100 80 GB: RunPod $1.19–1.59, Modal $2.50
  - L40S: $0.79–1.95
- **Default:** 1× H100 80 GB at about **$2.7–4.0/h**. Use A100 80 GB, about
  1.5–2× slower (estimate), when cost matters more than wall-clock time.
- **Throughput assumptions** (to be measured in Milestone 1 and used to
  re-estimate):
  - vLLM aggregate generation for a 4B model on one H100: 2–5k tokens/s at
    high batch
  - `fin_numeric` completions: about 1.5k tokens with thinking
  - `fin_tools` trajectories: about 3k generated tokens over about 8 turns

| Experiment | Size | GPU-hours (estimate) | Cost (estimate) |
|---|---|---|---|
| `fin_numeric` data generation | CPU; companyfacts bulk file | n/a | < $5 |
| Feasibility check, per model | 200 items × 16 samples ≈ 5M tokens | 0.5–1.5 | $2–6 |
| Base-model eval, full (test + control + post-cutoff + FinQA/TAT-QA, n = 4) | ≈ 30M tokens | 2–5 | $6–20 |
| Frontier API eval, per model | ≈ 5.5k items × n = 1–4 | n/a | $30–300 depending on model and n; compute from the price sheet at run time |
| RFT, one round (4B) | 5k prompts × 8 samples plus SFT | 5–10 | $15–40 |
| GRPO `fin_numeric` (4B) | 400–800 steps × 128 completions | 10–25 | $30–100 |
| GRPO `fin_numeric` (8B) | same | 20–50 | $60–200 |
| DPO (4B) | ~10k pairs | 2–4 | $6–16 |
| `fin_tools` DB build | CPU, per instance | n/a | < $5 |
| RFT `fin_tools`, one round (4B) | 3k prompts × 8 trajectories plus SFT | 10–20 | $30–80 |
| GRPO `fin_tools` (4B) | 300–500 steps × 64 trajectories | 20–60 (or 2–4 GPUs to cut wall-clock) | $60–240 |

**How the total is built** (GPU-hours, estimate):

| Line | Count × unit | Total |
|---|---|---|
| Feasibility checks | 3 models × 2 tasks × 0.5–1.5 | 3–9 |
| Base-model evals, both tasks | ~4 configs (4B/8B × thinking on/off) × 5–13 | 20–52 |
| RFT `fin_numeric` | 3 rounds × 5–10 | 15–30 |
| RFT `fin_tools` | 2 rounds × 10–20 | 20–40 |
| GRPO `fin_numeric` 4B, headline | 3 seeds × 10–25 | 30–75 |
| M4 ablations | 4 runs (`dr_grpo`, binary reward, LoRA rank 8/64) × 10–25 | 40–100 |
| GRPO `fin_numeric` 8B | 1 seed; 3 only if 8B is carried forward | 20–50 |
| DPO | 1 × 2–4 | 2–4 |
| GRPO `fin_tools` 4B, headline | 3 seeds × 20–60 | 60–180 |
| Test evals of trained models | ~10 × 2–5 | 20–50 |
| **Subtotal** | | **230–590** |
| Contingency for failed and debug runs | +30% | **~300–770** |

**Rough total through Milestone 6: about 300–800 GPU-hours (about
$0.8k–3.2k at $2.7–4.0/h), plus $200–1,000 of API spend** (estimate). Q10
sets the actual budget ceiling.

## 12. Evaluation protocol

**Frozen test splits.**

- Test files are identified by sha256 in their manifests.
- `core/eval.py` loads a `test*` split only with `--milestone <id>`. Every test
  access is written to `results/test_access.log`.
- Checkpoint selection, hyperparameters and prompt changes use `val` only.
- Test is evaluated once per method per milestone.

**Decoding.**

- Each model uses its model card's recommended sampling settings, fixed per
  model and recorded in the manifest.
- pass@1 is estimated from **n = 4** samples per item.
- *Alternative:* greedy decoding (n = 1). It is cheaper and deterministic, but
  Qwen3 thinking models are documented as degrading under greedy decoding,
  and one sample has high variance per item.
- pass@k up to 16 (64 for §6's analysis) uses a fixed 500-item subset.
- Max new tokens is fixed per task.

**Seeds.**

- Evaluation sampling seeds are {0, 1, 2, 3}.
- Training seeds: 1 seed for exploratory runs, 3 seeds for headline
  comparisons.
- Data-generation seeds are part of the data version.

**Statistics.**

- Report the mean with a 95% bootstrap CI over items.
- Method comparisons use a **paired** bootstrap on the same items.
- **Multiple training seeds** are combined with a **hierarchical bootstrap**:
  resample seeds with replacement, then items within each seed. The per-seed
  results are also shown as a range, so a single lucky seed is visible.
- This follows the bootstrap convention in coding-routing-benchmark
  (`benchmark/scoring.py`).

**Logged per run:**

- **Per item** (`predictions.jsonl.zst`): item ID, prompt hash, raw output (or
  trajectory), parsed answer, `total`, all components, input and output
  tokens, latency, cost, sampling seed.
- **Aggregates** (`metrics.json`): per split, per level, per template; all
  detector rates.
- **Training curves:**
  - reward mean and std
  - fraction of zero-variance groups
  - policy entropy
  - completion length and truncation rate
  - clip fraction, KL (if used) and grad norm
  - for `fin_tools`: tool calls per episode and error rates
- **Spot-check file** (§9).

**Cost per item.**

- API models: tokens × list price on the run date.
- Self-hosted: (GPU $/h × eval wall-clock hours) / items, at a stated
  hardware and concurrency.

**Latency.** p50 and p95 end-to-end per item at concurrency 1. Throughput in
items per minute is measured at concurrency 32. For `fin_tools`, latency
includes tool execution.

**Results table format** (`results/<task>/leaderboard.md`, generated;
illustrative, with no data):

| Method | Model | Grader (main / ext) | Test score (95% CI) | acc@0.5% / F1 | L1 | L2 | L3 | L4 | Control gap | FinQA | TAT-QA | $ / 1k items | p50 / p95 latency (s) | Train GPU-h | Run |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| base | Qwen3-4B (think) | @1.0.0 / ext@1.0.0 | — | — | — | — | — | — | — | — | — | — | — | 0 | link |
| frontier | (API model id) | @1.0.0 / ext@1.0.0 | — | — | — | — | — | — | — | — | — | — | — | n/a | link |
| RFT r2 | Qwen3-4B + LoRA | @1.0.0 / ext@1.0.0 | — | — | — | — | — | — | — | — | — | — | — | — | link |
| GRPO | Qwen3-4B + LoRA | @1.0.0 / ext@1.0.0 | — | — | — | — | — | — | — | — | — | — | — | — | link |

For `fin_tools`, the level columns become D1–D4. The control, FinQA and TAT-QA
columns become `test_ood_schema`, `test_ood_domain`, and robustness at
fault rate 0.3.

## 13. Reproducibility

Every run writes `results/<task>/<YYYYMMDD>-<method>-<model>-<shortsha>/`:

| File | Contents |
|---|---|
| `run.yaml` | the fully resolved config: task, split, model, sampler, trainer, LoRA, grader version |
| `manifest.json` | see below |
| `metrics.json` | aggregates (§12) |
| `predictions.jsonl.zst` | per-item outputs and scores |
| `spotcheck.md` | manual review (§9) |
| `train_log/` | trainer logs, or a pointer to the tracking run |

The fields in `manifest.json`:

- git sha, plus a dirty flag and the diff when dirty
- data version: split file sha256, generator git sha, data-generation seed, and
  source snapshot (e.g. `companyfacts.zip` date and sha256)
- model ids with Hugging Face revision hashes
- adapter sha256 and storage URI
- seeds: data, train, eval
- lockfile hash (`uv.lock`)
- hardware
- wall-clock time and cost

**Rules:**

- A run refuses to start on a dirty tree unless `--allow-dirty` is set.
  With the flag, the diff is stored.
- Adapters and large artifacts live outside git, in a private HF repo or an
  object store, and are referenced by hash.
- Every table in the docs and leaderboards is generated from `results/`, never
  typed by hand.

## 14. Repo layout

```
zen-rft/
├── core/                    # shared, framework-free (imports no trainer)
│   ├── task.py              # Item, Score, Task, Environment, MultiTurnTask, Trajectory (§4)
│   ├── numeric.py           # unit families, scale canonicalisation, decay curve: used by BOTH graders
│   ├── clients.py           # OpenAI-compatible clients (local vLLM, frontier APIs), with cost accounting
│   ├── eval.py              # run a model on a split; n samples; pass@k; test-access guard
│   ├── rft.py               # rejection-sampling loop: sample → score → filter → hand off to train/sft
│   ├── report.py            # bootstrap CIs, paired comparisons, leaderboard generation
│   ├── manifest.py          # run directory, manifest, hashes, git sha
│   └── spotcheck.py         # detector rates + sample selection for manual review
├── train/                   # thin entry points around existing trainers
│   ├── sft.py               # TRL SFT (RFT rounds, warm starts), LoRA
│   ├── grpo_single_turn.py  # Unsloth + TRL GRPOTrainer
│   ├── dpo.py               # TRL DPO (optional)
│   ├── multi_turn/          # prime-rl configs + launch script for verifiers environments
│   └── configs/             # YAML per experiment
├── tasks/
│   ├── fin_numeric/
│   │   ├── concepts.yaml    # canonical concept → candidate us-gaap tags
│   │   ├── xbrl.py          # companyfacts download and cache (User-Agent, ≤10 req/s)
│   │   ├── statements.py    # statement trees, identities, residual rows
│   │   ├── perturb.py       # perturbation scheme + separation/consistency checks
│   │   ├── render.py        # tables and text with row IDs, style variation
│   │   ├── templates.py     # question templates, paraphrases, reference programs
│   │   ├── generate.py      # split building, manifests
│   │   ├── grader.py        # score(); uses core/numeric.py
│   │   ├── task.py          # Task implementation
│   │   ├── vf_env.py        # verifiers adapter (eval, optional training)
│   │   └── tests/           # includes every worked example from the spec
│   └── fin_tools/
│       ├── build_db/        # logical schema, messiness transforms, physical writer, sources/
│       ├── configs/messiness/  # X0–X3 YAML
│       ├── tools.py         # the five tools, limits, error types
│       ├── faults.py        # fault injection
│       ├── env.py           # Environment implementation
│       ├── questions.py     # templates, reference SQL
│       ├── reference_solver.py  # scripted replay through real tools
│       ├── grader.py        # score(); uses core/numeric.py
│       ├── task.py
│       ├── vf_env.py        # verifiers adapter (the training path)
│       └── tests/
├── results/                 # run directories (small files in git), leaderboards, test_access.log
├── data/                    # gitignored: source caches, generated splits, DB instances
├── docs/
└── README.md
```

**What is shared and what is task-specific:**

- **Shared:** everything in `core/`, including numeric canonicalisation. A
  scalar answer in `fin_tools` is graded exactly like `fin_numeric`, so a
  "percent vs ratio" fix lands in both graders at once. `train/` entry points
  are shared too, and are parameterised by task name.
- **Task-specific:** data generation, rendering or environments, templates,
  graders, and verifiers adapters.
- **Isolation rule:** tasks never import each other. `fin_tools` reuses
  `fin_numeric`'s perturbation by calling `tasks.fin_numeric.perturb` through a
  small public function, which is the one allowed cross-task import. The
  alternative, moving perturbation into `core/`, is fine if a third task needs
  it.

## 15. Integration with other zen-tradings repos

**coding-routing-benchmark.**

What it does today:

- It measures whether a router picks the right tier among
  `claude-haiku` / `claude-sonnet` / `claude-opus` for coding prompts.
- Reference labels (`reference_model`) come from human labelling.
- The chosen model is never run.
- The model pool is fixed in code (`AVAILABLE_MODELS` in `benchmark/models.py`).
  Its README lists adding models as needing code changes.

zen-rft can add *outcome-grounded* labels:

1. Run the candidate pool over a fin task's `test` split with our graders. The
   pool is the trained LoRA(s), the base model, and the Claude tiers.
2. For each item, the label is the **cheapest model whose mean score over n
   samples ≥ τ**, for example 0.9.
3. Write those labels as a new prompt file in its schema: `id`, `category`,
   `prompt`, `reference_model`, and a `reference_rationale` that lists the
   per-model scores. A possible name is `prompts/fin_dev_v1.jsonl`.
4. Take per-model prices from our results into its `[prices.<model>]` config.

This needs two upstream changes: making the model pool configurable, and a
client for OpenAI-compatible endpoints. They are proposed as a PR there,
not done here. What this adds is that "right model" gets defined by measured
task success instead of judgement.

**zen-fundamentals** (design stage; RFCs 001–008).

- Its pipeline defines LLM roles: `planner`, `estimator`, `extractor`,
  `auditor` and `eval_judge`.
- `computed` claims must come from deterministic code with recorded formula
  and inputs.
- **Best fit: the `extractor` role.** A `fin_numeric` model locates figures in
  filings and returns cited rows and a formula over cell references.
  zen-fundamentals then recomputes the value in code. That fits its rule that
  arithmetic is never trusted to a model.
- **Second fit: the `auditor`.** Narrow yes/no checks such as "do these cited
  rows support this number", where zen-fundamentals requires a provider
  different from the estimator's. A self-hosted zen-rft model satisfies that
  by construction.
- **Caveat:** zen-fundamentals' first template is merger arbitrage, which
  means terms in S-4 / DEFM14A / 8-K filings, not 10-K metrics. We do not
  assume transfer. Using a zen-rft model there needs its own replay eval on
  their deal set.

**Serving a trained LoRA.**

| | Hosted provider | Self-hosted vLLM |
|---|---|---|
| Current state | Together has discontinued serverless LoRA; adapters run on dedicated endpoints. Fireworks does not support serverless deployment of LoRA models, but offers on-demand deployments, including multi-LoRA. | `--enable-lora --lora-modules name=path --max-loras N --max-lora-rank R` (the default max rank is 16, so it must be raised to our rank) |
| Billing | per deployment-hour (no pay-per-token for custom LoRA) | per GPU-hour, yours |
| Ops | managed scaling, uptime, security | you run it |
| Flexibility | limited to the provider's supported base models and ranks | any base, any rank, several adapters on one base |
| Data governance | prompts go to a third party | stays in our infra |

- **Trade-off:** hosted buys operations at a similar per-hour cost, because
  pay-per-token LoRA isn't available. vLLM is cheaper at steady load and
  serves every task adapter from one base model, but idle GPUs cost money and
  we would be on call.
- **Recommendation:** use self-hosted vLLM with multi-LoRA for evaluation and
  internal pilots. Revisit a hosted dedicated deployment when an external
  consumer needs an SLA.
- *Alternative:* merge the adapter into the base weights. That gives the
  simplest serving and slightly lower latency, but costs one full model copy
  per task.

## 16. External facts and sources

All checked 2026-09-24. Re-verify before relying on them, because several
changed during 2025–2026.

| Fact | Source |
|---|---|
| verifiers v1 (`Taskset`, `vf.Task`, `@vf.reward`) and deprecated legacy v0 envs/rubrics; prime-rl LoRA, single-GPU, loss mask | docs.primeintellect.ai/verifiers (v1 overview, legacy environments/training), docs.primeintellect.ai/prime-rl |
| ART: `TrainableModel`, `Trajectory`, `LocalBackend` (vLLM + Unsloth), GRPO, LoRA only, v0.5.19 | github.com/OpenPipe/ART, art.openpipe.ai |
| verl agent loop `response_mask`, LoRA, repo moved to verl-project | verl.readthedocs.io (agent_loop, sglang multiturn, ppo_lora) |
| TRL `loss_type` options incl. `dapo` (default) and `dr_grpo`; `scale_rewards`; `mask_truncated_completions`; `environment_factory`, `tool_mask` | github.com/huggingface/trl `grpo_config.py`, huggingface.co/docs/trl grpo_trainer |
| Dr. GRPO removes length and std normalisation | arXiv 2503.20783 |
| DAPO: Clip-Higher, dynamic sampling, token-level loss, overlong shaping | arXiv 2503.14476 |
| Qwen3-4B/8B licence and context; Qwen3-2507; Qwen3.5 small models | huggingface.co/Qwen/Qwen3-4B, huggingface.co/Qwen/Qwen3.5-4B |
| SEC companyfacts, frames and bulk endpoints; 10 req/s; declared User-Agent | sec.gov/search-filings/edgar-application-programming-interfaces |
| FinQA and TAT-QA split sizes and licences | github.com/czyssrs/FinQA, github.com/NExTplusplus/TAT-QA |
| Together / Fireworks LoRA serving policies; vLLM LoRA flags | docs.together.ai/docs/lora-inference, docs.fireworks.ai/fine-tuning/deploying-loras, docs.vllm.ai/en/latest/features/lora.html |
| GPU prices | runpod.io/pricing, lambda.ai/pricing, modal.com/pricing |

Cited from memory and **not re-checked in this pass**: Yue et al. (2025) on
RLVR and pass@k; Thinking Machines "LoRA Without Regret" (2025); Chen et al.
(2021) for the unbiased pass@k estimator. Verify them before external use.
