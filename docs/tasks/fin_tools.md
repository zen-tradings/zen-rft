# Task spec: `fin_tools`

Multi-turn tool use over a deliberately messy financial database.

Status: design. None of this is implemented yet, and no results exist.

- Every number marked *target* or *initial value* is a guess that we will
  revisit when the environment is first built (Milestone 5 in
  [`../milestones.md`](../milestones.md)).
- **All numeric parameters in this spec are initial values unless stated
  otherwise.** That covers limits, caps, weights, rates, thresholds and sizes.
- Messiness levels are named **X0–X3**, so they don't collide with milestone
  names M1–M7.

Contents:

1. [What the model does](#1-what-the-model-does)
2. [Data sources](#2-data-sources)
3. [Schema generation and messiness](#3-schema-generation-and-messiness)
4. [Tools](#4-tools)
5. [Questions: templates and difficulty](#5-questions-templates-and-difficulty)
6. [Rollout](#6-rollout)
7. [Grader](#7-grader)
8. [Fault injection](#8-fault-injection)
9. [Splits and the generalisation test](#9-splits-and-the-generalisation-test)
10. [Training details specific to this task](#10-training-details-specific-to-this-task)
11. [Reward hacking: exploits and guards](#11-reward-hacking-exploits-and-guards)
12. [Worked examples](#12-worked-examples)

---

## 1. What the model does

The model gets a natural-language question about funds, securities, prices or
fundamentals, plus five tools over a SQLite database of 30–400 tables (target, depending
on messiness level) with inconsistent naming.

- It explores the schema, queries, and recovers from errors.
- The episode ends when it replies with a final JSON answer and no tool call,
  or when it hits the step limit.
- The grader compares the answer with a value computed from the database when
  the question was generated.

## 2. Data sources

| Logical domain | Source | Real or synthetic |
|---|---|---|
| Companies, identifiers, SIC codes, filings index | SEC EDGAR submissions and company facts | real metadata |
| Fundamentals (annual, quarterly) | SEC XBRL company facts, perturbed with the `fin_numeric` scheme (§5 of that spec) | real, then perturbed |
| Daily prices, corporate actions | **source to be decided** (Q1 in open questions) | real then rescaled, or synthetic |
| Institutional holdings | SEC Form 13F data sets | real |
| Internal fund positions, lots, transactions | simulated funds trading the real (rescaled) securities | **synthetic** |
| Filing snippets for `search_filings` | generated from templates using the perturbed DB values | **synthetic** |

**Synthetic internal funds.** Cost basis, and therefore unrealised P&L, is not
public: 13F filings report only value and shares. So internal fund books are
simulated.

- 12–30 funds per DB instance (target), each with a style rule, e.g.
  "large-cap momentum" or "value, sector-capped".
- Rebalancing is monthly, with lot-level transactions at the real (rescaled)
  closing prices.
- This produces realistic cost bases, realised and unrealised P&L, and
  position histories. Every answer stays exactly computable.

**Filing snippets are synthetic.** Real filing text contains real,
unperturbed numbers that would contradict the perturbed fundamentals.

- *Alternative:* index real filing text with numbers masked. That is more
  realistic, but the masking is imperfect and leaks unperturbed figures (Q9).

**Prices.** Each security's price path is multiplied by a constant factor
`s_sec ~ LogUniform(0.3, 3.0)`. This preserves returns and splits while
breaking recall of price levels. *Initial value.*

**13F caveat.** The SEC changed the 13F value field from thousands of dollars
to whole dollars for filings from January 2023. This is to be verified during
the build. We keep it as a *real* unit inconsistency and do not normalise it. 13F value
fields are multiplied by the same `s_sec` as prices, so they don't reveal real
price levels.

## 3. Schema generation and messiness

### 3.1 Two layers

1. **Logical schema** (hidden, about 25 entities). Examples:
   - `company`, `security`, `ticker_history`
   - `fundamental_fact`, `filing`
   - `price_daily`, `corporate_action`
   - `fund`, `fund_position_snapshot`, `fund_lot`, `fund_transaction`
   - `inst_holding_13f`, and a few more
   - Canonical names, clean types, one ID scheme.
2. **Physical schema** (visible to the model). A seeded generator applies
   *messiness transforms* to the logical schema and writes the SQLite file. It
   also writes a hidden `mapping.json` (physical ↔ logical) and a hidden clean
   copy, `truth.db`.

Answers are computed on `truth.db` with reference SQL over the logical schema.
Then a **scripted reference solver** replays a solution through the real tools
on the physical DB, within the real limits, and must reproduce the answer.
Items that fail this replay are dropped. This guarantees that every question
can be solved with the tools as specified.

### 3.2 Messiness transforms

Each transform has a rate knob in `tasks/fin_tools/configs/messiness/*.yaml` (`x0.yaml` … `x3.yaml`).

| Transform | Example | Knob |
|---|---|---|
| Naming style per table family | `pos_snap_2024q3`, `PositionSnapshot`, `TBL_POS_SNAP`, `fs_pos_snap` | `naming_entropy` ∈ [0,1] |
| Column abbreviations and synonyms | ticker → `tkr` / `symbol` / `sec_cd`; date → `asof_dt` / `trd_dt` / `AsOfDate` | `abbrev_rate` |
| Time partitioning | `px_eod_2019` … `px_eod_2025`; quarterly position tables | `partition_families` |
| Decoy copies | `_bak`, `_old`, `_v2`, `_tmp`, `stg_` tables; some stale, some with subtly different values | `decoy_rate`, `decoy_staleness` |
| Mixed ID schemes | CIK as int vs zero-padded text, CUSIP, internal `sec_id`, point-in-time tickers | `id_scheme_mix` |
| Units | amounts in thousands in one family, units in another; noted in comments only sometimes | `unit_mismatch_rate` |
| Date formats | ISO text, `YYYYMMDD` text, epoch int | `date_format_mix` |
| Encoded values | side `B`/`S` vs `1`/`-1`, status codes `A`/`I` | `code_rate` |
| Nulls and sentinels | `NULL`, `''`, `-999`, `'N/A'` | `sentinel_rate` |
| Wide vs long | fundamentals as long `(tag, value)` rows and as wide per-statement-per-year tables | `wide_long_mix` |
| Soft deletes and validity windows | `is_del`, `valid_from` / `valid_to` | `scd_rate` |
| Irrelevant tables | `etl_job_log`, `user_prefs`, `report_cache_*` | `noise_tables` |
| Documentation coverage | `meta_table_catalog` and column comments describe only some tables, and mark authoritative vs deprecated | `doc_coverage` ∈ [0,1] |

**Messiness levels** (initial values):

| Level | naming_entropy | decoy_rate | unit_mismatch | doc_coverage | Tables (target) |
|---|---|---|---|---|---|
| X0 clean | 0 | 0 | 0 | 1.0 | ~30 |
| X1 | 0.3 | 0.1 | 0.1 | 0.8 | ~120 |
| X2 | 0.6 | 0.2 | 0.25 | 0.5 | ~250 |
| X3 | 0.9 | 0.3 | 0.4 | 0.3 | ~400 |

- X0 is for debugging only.
- Training mixes X1–X3.
- *Alternative:* a single fixed messy schema, which is simpler. The model
  would then learn *that* schema rather than how to explore one, and the
  unseen-table test (§9) would be meaningless.

**Size.** The target is about 1,000 securities with 2010–2025 daily prices,
which is roughly 4M price rows. That puts one DB instance at about 0.3–1 GB.
All instances are rebuilt deterministically from `(seed, messiness config,
source snapshot hashes)`.

## 4. Tools

**Common behaviour:**

- Every tool returns a JSON string: `{"ok": true, "result": ...}` or
  `{"ok": false, "error": {"type": "...", "message": "..."}}`.
- Output is capped at **8,000 characters** (initial value). A result that goes
  over is truncated at a row boundary with `"truncated": true`.
- Tool schemas are exposed in OpenAI function-calling format.
- **Errors every tool can return:**

| Error type | When | Counted as |
|---|---|---|
| `InvalidToolCall` | unknown tool name, arguments that aren't valid JSON, or a schema violation (missing, extra or ill-typed argument) | `p_invalid` |
| `ServiceUnavailable` | injected fault only: `database temporarily unavailable; retry` | exempt |
| `InvalidArgument` | the call is well-formed but a value is out of range (tables below) | model error, so it can count toward `p_unrecovered` |

- **Why these limits:** the caps (8k chars, 500 rows, 5 s-equivalent) are
  sized so the reference solver never hits them. They are initial values.
  Milestone 5 logs how often trained models hit each cap, and a cap is
  loosened if legitimate solutions hit it.

### `list_tables`

```python
def list_tables(pattern: str | None = None, limit: int = 100, offset: int = 0) -> str:
    """List tables whose name matches a SQL LIKE pattern (case-insensitive).

    Returns [{"name", "approx_rows", "comment"}] sorted by name. The comment is
    the catalog description, or null when the table is undocumented.
    limit is capped at 100. Use offset to page through the results.
    """
```

| Error type | When | Message |
|---|---|---|
| `InvalidArgument` | limit < 1 or > 100, offset < 0 | `limit must be between 1 and 100` |

### `describe_table`

```python
def describe_table(table: str, sample_rows: int = 3) -> str:
    """Return columns (name, declared type, nullable, comment), primary key,
    declared foreign keys, and up to `sample_rows` example rows (max 5).
    """
```

| Error type | When | Message |
|---|---|---|
| `UnknownTable` | the table does not exist | `no such table: 'pos_snapshot'` |
| `InvalidArgument` | sample_rows outside 0–5 | `sample_rows must be between 0 and 5` |

"Did you mean …" suggestions are **off** by default (`suggest_on_unknown:
false`). Real databases don't offer them.

*Alternative:* turn them on for X0–X1. That makes early training easier, but
teaches the model to lean on hints.

### `run_sql`

```python
def run_sql(query: str, max_rows: int = 200) -> str:
    """Execute ONE read-only SELECT/WITH statement.

    Returns {"columns", "rows", "row_count", "truncated", "elapsed_ms"}.
    Results are cut off at min(max_rows, 500) rows and at the 8,000-character
    cap.
    """
```

**Limits and enforcement:**

- **Read-only, enforced in three ways:**
  - a `file:…?mode=ro` URI
  - `PRAGMA query_only = ON`
  - a `sqlite3` authorizer callback that allows only `SQLITE_SELECT`,
    `SQLITE_READ`, `SQLITE_FUNCTION` and `SQLITE_RECURSIVE` (for recursive
    CTEs), and denies everything else, including `ATTACH`, `PRAGMA` and any
    write
- **Separate connections.** The authorizer is installed only on `run_sql`'s
  connection. `describe_table` and `list_tables` use their own internal
  read-only connection, because they need `PRAGMA table_info`.
- **Size guard.** `setlimit(SQLITE_LIMIT_LENGTH, 1_000_000)` guards against
  `zeroblob` or `printf` memory blowups.
- **Time limit.** A deterministic **VM-instruction budget**, enforced by a
  progress handler that counts opcodes, calibrated to about 5 s on reference
  hardware. A 30 s wall-clock backstop also applies. Rewards therefore don't
  depend on machine load.
- **One statement per call.**
- **Error mapping:**
  - SQLite "not authorized" → `ReadOnlyViolation`
  - Python "You can only execute one statement at a time" →
    `MultipleStatements`
  - "interrupted" → `Timeout`

| Error type | When | Message |
|---|---|---|
| `SQLError` | SQLite parse or runtime error | the SQLite message, verbatim, e.g. `no such column: tkr` |
| `ReadOnlyViolation` | any non-SELECT statement | `only SELECT or WITH statements are allowed` |
| `MultipleStatements` | more than one statement | `submit exactly one statement per call` |
| `Timeout` | instruction budget exceeded | `query exceeded 5.0s and was cancelled; add filters or LIMIT` |
| `InvalidArgument` | max_rows outside 1–500 | `max_rows must be between 1 and 500` |

### `get_price_history`

```python
def get_price_history(ticker: str, start: str, end: str,
                      field: Literal["close", "adj_close", "open", "high", "low", "volume"] = "close",
                      frequency: Literal["daily", "weekly", "monthly"] = "daily") -> str:
    """Price series for a ticker as it was known on `end` (point-in-time
    ticker resolution). Dates are ISO YYYY-MM-DD. Returns [{"date", "value"}].
    Max 400 points per call. Uses the same (rescaled) data as the price tables.
    """
```

This models a separate internal pricing service. It is a second route to the
same data, so questions have more than one valid path.

| Error type | When | Message |
|---|---|---|
| `UnknownTicker` | the ticker did not resolve on `end` | `ticker 'XYZ' not found as of 2024-09-30` |
| `InvalidDateRange` | start > end, or a date is badly formatted | `start must be <= end, format YYYY-MM-DD` |
| `RangeTooLarge` | more than 400 points | `requested 1,260 points; max 400. Narrow the range or use weekly/monthly` |

### `search_filings`

```python
def search_filings(query: str, company: str | None = None, form: str | None = None,
                   date_from: str | None = None, date_to: str | None = None, k: int = 5) -> str:
    """BM25 search (SQLite FTS5) over filing snippets. `company` accepts a
    ticker, CIK, or name. Returns up to k (max 10) hits:
    {"accession", "cik", "company", "form", "filed", "section", "snippet"},
    with snippets of at most 600 chars.
    """
```

| Error type | When | Message |
|---|---|---|
| `UnknownCompany` | the company filter did not resolve | `company 'Acme' not found` |
| `InvalidArgument` | k outside 1–10, bad date | `k must be between 1 and 10` |

**Why these five tools.** They separate schema discovery (`list_tables`,
`describe_table`) from data access (`run_sql`). They also offer one
non-SQL data path (`get_price_history`) and one text path (`search_filings`,
used for things like former names, segment names, and disclosed charges).

*Alternative:* a single `run_sql` tool, with the model reading
`sqlite_master` itself. That is simpler, but mixing discovery with querying
makes invalid-call statistics harder to interpret.

## 5. Questions: templates and difficulty

**Answer types:**

- `scalar`: a number with unit and scale, graded as in `fin_numeric`
- `entity_list`: an unordered set of securities, companies or funds
- `ranked_entity_list`: top-k. Graded as a set in v1; order is a diagnostic
- `entity_values`: a list of (entity, number) pairs
- `date` / `string`: exact match after normalisation

**Difficulty** is defined by what the reference SQL touches. Every item stores
`n_tables`, `n_joins` and `traps` (the list of messiness features it
requires).

| Level | Tables / joins | Typical traps | Reference tool calls (target) |
|---|---|---|---|
| D1 | 1 table, 0 joins | naming, date format | 3–4 |
| D2 | 2 tables, 1 join | ID mismatch, units | 4–6 |
| D3 | 3 tables, 2 joins, aggregation | partitions, decoys, point-in-time | 6–9 |
| D4 | ≥4 tables and ≥2 traps | ticker changes, unit families, validity windows, stale decoys | 8–12 |

**Templates v1** (initial; about 16):

| ID | Level | Question pattern | Answer type | Splits |
|---|---|---|---|---|
| T1-01 | D1 | Closing price of {ticker} on {trading date} | scalar | all |
| T1-02 | D1 | How many positions did fund {F} hold as of {date}? | scalar | all |
| T1-03 | D1 | How many transactions did fund {F} execute in {month}? (synthetic data) | scalar | all |
| T2-01 | D2 | {concept} for {ticker} in fiscal {year}, as reported in that year's 10-K | scalar | all |
| T2-02 | D2 | Top {k} positions of fund {F} by market value as of {date} | ranked_entity_list | all |
| T2-03 | D2 | Total dividends per share of {ticker} with ex-dates in {year} | scalar | ood_domain only |
| T2-04 | D2 | Which funds held {ticker} as of {date}? | entity_list | all |
| T3-01 | D3 | Largest {k} unrealised losses in fund {F} as of {date}, with amounts (negative USD) | entity_values | all |
| T3-02 | D3 | Positions of fund {F} as of {date} whose latest-FY revenue growth (10-K filed before {date}) exceeded {x}% | entity_list | all |
| T3-03 | D3 | Of fund {F}'s top {k} holdings by market value as of {quarter end}, which were also reported by 13F filer {M} for that quarter? | entity_list | ood_domain only |
| T3-04 | D3 | Market value of fund {F} as of {date} | scalar | all |
| T4-01 | D4 | Total unrealised P&L of fund {F}'s positions in companies that reported a net loss in their latest 10-K filed before {date} | scalar | all |
| T4-02 | D4 | Realised P&L of fund {F} in {year}, average-cost method | scalar | all |
| T4-03 | D4 | Split-adjusted price return of {ticker} from {d0} to {d1}, crossing a year partition, from the corporate-actions table | scalar | ood_domain only |
| T4-04 | D4 | Fund {F}'s positions whose ticker changed between {d0} and {d1}, with current tickers | entity_list | all |
| T4-05 | D4 | Using the restructuring charge disclosed in {company}'s {year} 10-K, adjusted operating margin = (operating income + charge) / revenue | scalar | all |

Two templates, chosen before generation, are held out of `train`. They appear
only in `test_heldout_template` (§9).

**Conventions** stated in every system prompt, so the questions are not
ambiguous:

- All internal fund amounts, costs and prices are USD. There is no FX in v1.
- Losses and negative P&L are reported as **negative** numbers.
- Fundamentals are "as reported in the 10-K (or 10-Q) for that period", not
  later restatements.
- Dividends are assigned to periods by ex-date. Returns are price returns,
  split-adjusted, excluding dividends.
- Dates in questions that ask for a price on a single day are always trading
  days.

- "as of D" means the latest snapshot on or before D, and the last trading day
  on or before D.
- Prices are unadjusted closes unless the question says otherwise.
- Unrealised P&L = qty × (close − average cost).
- Entities may be named by ticker (point-in-time), CUSIP, CIK or internal ID.

**Generation filters.** An item is rejected if any of these hold:

- the answer is empty
- the answer has more than 25 entities
- the reference solver needs more than 12 calls
- the result would exceed the tool output caps under the reference query
- the answer is identical on a decoy table *and* on the authoritative table.
  Such an item cannot test the decoy trap, so it is retagged without that trap.

## 6. Rollout

```
obs = env.reset(item, seed)               # system prompt + question + tool schemas
for turn in range(max_turns):             # max_turns = 24 (initial value)
    msg = policy(obs)                     # assistant turn: text and/or tool calls
    if no tool calls in msg:              # final answer
        break
    obs = env.step(msg)                   # execute calls (with fault injection), append tool messages
    if env.tool_calls_used >= 20:         # hard tool-call limit (initial value)
        obs = env.final_turn_notice()     # calls beyond 20 in a turn are not executed;
        msg = policy(obs); break          # the model gets one tool-free turn to answer
score = task.score(item, task.parse(env.trajectory()))   # parsed answer carries tool_events
```

**Limits and answer format:**

- **Output limits:** 2,048 new tokens per assistant turn, and a total context
  target of 24k tokens. Both are initial values.
- **Final answer:** an assistant message with no tool call. The instructions
  ask for exactly one fenced `json` block. As in `fin_numeric`, the parser
  takes the **last** block. Example:
  `{"answer_type": "entity_values", "answer": [...], "unit": "USD", "scale": "units"}`.
- A message with no tool call and no parseable JSON ends the episode with
  correctness 0.
- *Alternative:* a dedicated `submit_answer` tool. It gives cleaner
  termination, but it is a sixth tool the model must learn. Ending on "no tool
  call" also matches how verifiers' tool environments and TRL's tool loop
  already terminate.

## 7. Grader

`score(item, parsed) -> {total, components, info}`, where `parsed = parse(trajectory)` holds the final answer and the list of `ToolEvent`s (design §4.3).

### 7.1 Correctness `C ∈ [0, 1]`

**Entity canonicalisation.**

- Every predicted entity string is resolved to a canonical entity ID through
  the hidden mapping. Tickers are resolved point-in-time at the question's as-of
  date; CUSIP, CIK and `sec_id` also resolve.
- Unresolvable strings count as wrong predictions.

**Grounding gate.** A predicted entity counts as grounded only if it appeared,
by some identifier, in a **result cell of a successful data-returning call**
before the model's own text or tool arguments first mentioned it.

- Numeric IDs (`sec_id`, CIK) match only in columns that `mapping.json` marks
  as ID columns. This stops matches on incidental numbers such as quantities.
- Ungrounded entities count as unmatched. So echoing a guess back through
  `SELECT 'NWCI'` or an error message does not work.
- *Alternative:* no gate, with the "ungrounded" rate logged as a detector
  only. That is simpler, but it pays for recall and lucky guesses.

**No-data rule.** If the trajectory has zero successful data-returning calls
(`run_sql` returning rows, `get_price_history`, `search_filings`), then
`C = 0` for every answer type, scalars included.

**Canonicalisation and dedupe.** Predictions are deduplicated *after*
canonicalisation, so a ticker and an internal ID for the same entity count
once.

**Scoring by answer type.**

| Answer type | Correctness |
|---|---|
| `scalar` | `C = sign_ok · scale_ok · numeric`, using the `fin_numeric` curve, floors and gates (§8.1–8.3). The citation factor is dropped, not zeroed, so the maximum is 1.0. |
| `entity_list`, `ranked_entity_list` | F1 between predicted and gold canonical sets. Rank correlation is logged for ranked lists. |
| `entity_values` | **soft F1**: for matched entities, `m = Σ numeric(r_e)`; precision = m / \|pred\|, recall = m / \|gold\|, then C = harmonic mean |
| `date`, `string` | 1 if equal after normalisation (ISO dates, case-folded names via the company alias table), else 0 |

*Alternative for lists:* exact set match, which is simpler and stricter.
Overlap gives partial credit on long lists, where one miss shouldn't zero out
the whole answer.

*Alternative for `entity_values`:* score entity F1 and value accuracy as two
separate components. That makes diagnosis easier, but needs a weighting
between them. Soft F1 folds both into one number with no extra weight.

### 7.2 Penalties (only the model's own mistakes)

| Penalty | Definition | Per event | Cap |
|---|---|---|---|
| `p_invalid` | `InvalidToolCall`: unknown tool name, arguments that aren't valid JSON, or a schema violation (missing, extra or ill-typed argument) | 0.05 | 0.15 |
| `p_unrecovered` | a model-caused error (`SQLError`, `UnknownTable`, `InvalidArgument`, a model-caused `Timeout`, etc.) that is **not recovered** (definition below) | 0.05 | 0.10 |
| `p_budget` | each tool call beyond the budget `B = min(ceil(1.5 × ref_calls) + 2, 16) + n_injected_faults` | 0.02 | 0.10 |

**Recovered.** An error counts as recovered only if a *later successful call
to the same tool* touches at least one table or entity that the failed call
referenced. For `run_sql`, the tables come from the query. This way a junk
`SELECT 1` does not clear an error.

**Budget cap.** The cap at 16 keeps `p_budget` reachable for D4 items, which
would otherwise have a budget at or above the 20-call hard limit.

*Alternative for the budget:* a fixed budget per level, e.g.
D1 6 / D2 9 / D3 12 / D4 16. It is simpler, but ignores per-item variation
in how many steps the reference solution needs.

- Injected faults (§8) never count as invalid or unrecovered.
- A model-caused error followed later by a successful call is **not**
  penalised. Recovering is the behaviour we want.

### 7.3 Total

```
P     = min(0.30, p_invalid + p_unrecovered + p_budget)
total = C · (1 − P)
```

**Why multiplicative, and why these weights.**

- **Penalties never push the model toward guessing.** Because they scale
  `C`, a wrong answer scores 0 whether the model used 3 tool calls or 20. The
  penalties only rank *successful* trajectories against each other.
  - An additive penalty (`C − P`, which can go negative) would make "answer
    immediately without tools" the safe choice whenever the model is unsure.
    That is the behaviour we least want.
- **The budget penalty is small** (at most 0.10), and the budget is
  generous: 1.5× the scripted solver plus 2, extended by every injected fault.
  - Exploring beats guessing whenever `C_explore · 0.9 > C_guess`, i.e.
    whenever extra calls raise expected correctness by more than about 11%
    relative.
  - `C_guess = 0` by construction, because of the no-data rule and the
    grounding gate. Without those rules, guessing would still be near zero for
    templates over synthetic fund data or perturbed fundamentals. So the
    penalty only discourages real waste, such as repeated identical queries
    or table dumps.
- **The invalid-call penalty** (0.05 each) makes malformed calls cost about
  as much as a small numeric error. It is enough to discourage sloppy argument
  formats without making the model avoid tools.
- *Alternative:* no budget penalty. Nothing would then discourage 20-call
  wandering. Group-relative advantages already favour shorter successful
  trajectories only weakly.

### 7.4 No substantial reward for intermediate steps

Intermediate steps get **zero reward in v1**. Process metrics are logged
only: whether the model touched the gold tables, whether it used a decoy,
whether it found the catalog, and how many schema-discovery calls it made.

Reasons:

1. **Many valid paths.** A question can be answered through `run_sql` on the
   price tables or through `get_price_history`, and through the long or the
   wide fundamentals table. Rewarding a specific step would penalise equally
   good strategies.
2. **Process rewards get gamed.** If "called `describe_table` on the gold
   table" pays, the model learns to make that call ritually.
3. **The outcome is exactly verifiable.** Unlike open-ended agent tasks, we
   know the answer, so outcome reward carries the full signal.
4. **GRPO assigns credit at the trajectory level anyway.** A step bonus mainly
   adds noise to within-group comparisons.

*Fallback, only if the feasibility check shows almost no successes:* a
temporary shaping term of at most 0.05 for touching a gold table, used in
early curriculum only and removed before the headline runs. The trade-off is
faster early learning against the risk of teaching ritual calls.

### 7.5 Components returned

`correctness`, `answer_parsed`, `answer_type_ok`, `precision`, `recall`,
`numeric_mean`, `grounding_dropped`, `n_tool_calls`, `budget`, `n_invalid`,
`n_model_errors`, `n_unrecovered`, `n_injected_faults`, `n_fault_recovered`,
`used_decoy_table`, `touched_gold_tables`, `hit_step_limit`, `P`, `total`.

**Grader version:** `fin_tools/grader@1.0.0`.

## 8. Fault injection

**Configuration** (initial values):

```yaml
faults:
  rate: 0.10                 # per tool call (training default)
  per_tool_rate: {}          # optional overrides, e.g. run_sql: 0.15
  types:
    transient: 0.4           # {"type":"ServiceUnavailable","message":"database temporarily unavailable; retry"}
    timeout: 0.4             # same type and message as a real timeout of that tool (e.g. run_sql: "query exceeded 5.0s ...")
    partial: 0.2             # rows returned, plus "truncated": true and a "partial_result": "transient failure; retry for full result" flag
  max_per_episode: 3
  seed_by: [item_id, group_seed, call_index]   # u_k = hash(...) in [0,1); fault fires if u_k < rate(tool of call k)
```

- **Faults are always explicit.** A fault never silently corrupts data. The
  model must be *able* to tell that something went wrong, otherwise the task is
  unfair.
- **Faults are drawn from `u_k = hash(item_id, group_seed, k)`.** They are
  shared by all rollouts in a GRPO group and change across epochs, because
  `group_seed` changes. So the schedule cannot be memorised.
  - Rollouts diverge, so call k may be a different tool or query in each
    sample. The shared draws therefore *reduce* fault luck within a group
    rather than removing it.
  - *Alternative:* per-rollout seeding. That covers more fault patterns per
    item, but adds more reward variance unrelated to behaviour.
- **Injected timeouts look exactly like real ones.** A model can't tell an
  injected timeout from a heavy query. That is realistic, and the right move
  is the same either way: retry once, then narrow the query.
  - *Alternative:* a distinct message. It is easier to learn from, but teaches
    the model to trust the message text.
- **`partial` faults** return rows with `"ok": true, "truncated": true,
  "partial_result": ...`. They are logged as `kind = "injected"`.
- **Reward for completing anyway.** A trajectory that hits injected faults and
  still answers correctly earns the full `C`, because:
  - injected faults are exempt from `p_invalid` and `p_unrecovered`
  - the budget grows by one per injected fault

  No extra bonus is paid. A bonus would make scores incomparable across fault
  rates.
- **Evaluation** runs at fault rates {0, 0.1, 0.3}.
  - We report `robustness = success(rate=0.3) / success(rate=0)`.
  - We also report `fault_recovery_rate`, the share of injected faults followed
    by a successful retry.

## 9. Splits and the generalisation test

**DB instances:**

- The generator produces separate instances from different seeds.
- **Training instances:** 6, at X1–X3 (target).
- **OOD-schema instances:** 2. They are built from a **held-out naming
  vocabulary**: different abbreviation dictionaries, style mix and partition
  granularity. None of their physical table names occur in any training
  instance.
  - Their fund books and price scale factors are regenerated as well. Answers
    memorised during training therefore don't transfer, and only schema
    navigation does.
- **Training instances** contain no 13F table and no corporate-actions table.
  Adjusted-close columns remain, which is a small, accepted leak of
  corporate-action information.

**Splits** (all sizes are targets):

| Split | Size (target) | DB instances | Tests |
|---|---|---|---|
| `train` | 6,000 | 6 training instances | n/a |
| `val` | 500 | training instances | checkpoint selection |
| `test_iid` | 1,000 | training instances, unseen questions (new parameter draws of training templates) | unseen questions |
| `test_heldout_template` | 300 | training instances, 2 templates never trained on | unseen question types |
| `test_ood_schema` | 500 | 2 OOD-schema instances with **regenerated fund books and `s_sec`** | tables never seen in training |
| `test_ood_domain` | 300 | 1 separate instance that includes the table families **absent from all training instances**: 13F holdings, corporate actions. Uses templates T2-03, T3-03, T4-03 | new domains |

- **Level mix:** D1 15% / D2 30% / D3 35% / D4 20%.
- **No shared question instance:** a question never appears in two splits.
- *Alternative for the level mix:* a uniform mix. It gives more signal on D4,
  but puts too many unsolvable items early in training. The skew toward
  D2–D3 matches where the base model is expected to be learnable. This gets
  re-weighted after the M5 feasibility check.

## 10. Training details specific to this task

- **Loss masking.** Tool-result tokens, and the system and user tokens, are
  masked out of the loss. The loss covers assistant tokens only: reasoning,
  tool-call JSON and the final answer.
  - verifiers/prime-rl, verl's agent loop (`response_mask`), and TRL's tool
    loop (`tool_mask`) all do this.
  - The chosen framework must be checked by a unit test: render one
    trajectory and assert that the mask covers exactly the assistant spans.
- **Thinking across turns.** Qwen3 chat templates drop earlier `<think>`
  blocks when re-rendering history. That breaks token-level consistency
  between rollout and training.
  - We need a framework that trains on the tokens actually generated
    (token-in/token-out), not on a re-rendered transcript. See the framework
    comparison in design §8.
- **Curriculum.** Start at D1–D2 and X1, then add D3–D4 and X2–X3 once D2
  success passes 60% on val (initial threshold).

## 11. Reward hacking: exploits and guards

| Likely exploit | Guard |
|---|---|
| Guess entities without querying | Grounding gate: entities never seen in tool output are unmatched |
| List every holding to max out recall | F1 penalises precision |
| Dump whole tables and read off answers | Row cap of 500, 8k-char output cap, budget penalty; the time limit stops huge scans |
| Write to or corrupt the DB, `ATTACH` other files | Read-only URI, `query_only`, authorizer deny-list, a fresh connection per rollout, an immutable file (checksum checked after each batch) |
| Reach the hidden `truth.db` or `mapping.json` | The environment process opens only the physical DB path. There are no filesystem tools, and the truth files live outside the sandbox mount |
| Mine error messages for data | Errors contain schema names and SQLite messages only, never cell values. Suggestions are off |
| Retry loops to burn budget on injected faults | Budget grows by exactly one per injected fault; `max_per_episode` caps faults |
| Several JSON blocks or hedged answers | Only the last block counts; answer type must match |
| Right answer from a decoy table by luck | Generation filters out items where decoy and authoritative give the same answer (§5) |
| Memorised real prices or fundamentals | Price rescaling, fundamental perturbation, synthetic fund books |

**Spot checks.** Use the procedure in design §9. For this task, the reviewer
reads the full trajectory and checks four things:

- (a) the SQL actually computes the answer
- (b) the tables used are authoritative
- (c) the recovery after a fault was real, not a coincidence
- (d) there are no degenerate patterns, such as identical repeated calls

## 12. Worked examples

The fund, tickers and figures are **illustrative and fictional**. The
tickers are invented, and the numbers were computed from the values shown.

### Example A: T3-01 (D3), with one injected fault

**Question:** For fund GLB03 (ZEN Global Opportunities), as of 2024-09-30,
which three positions had the largest unrealised losses, and how large was
each loss in USD? Use unadjusted closing prices on the as-of date and each
position's average cost.

**Hidden gold**, computed on `truth.db`:

| Entity | qty | avg cost | close 2024-09-30 | Unrealised P&L |
|---|---|---|---|---|
| NWCI | 12,000 | 84.20 | 61.35 | −274,200 |
| TLVQ | 30,000 | 18.75 | 11.02 | −231,900 |
| BRKX | 5,500 | 212.10 | 198.40 | −75,350 |
| (not in top 3) OAKQ | 15,000 | 33.60 | 29.95 | −54,750 |

Metadata: `n_tables = 3` (`pos_snap_2024q3`, `px_eod_2024`, `SEC_MASTER`),
`n_joins = 2`, `traps = [partition, decoy (pos_snap_2024q3_bak), date_format]`,
`ref_calls = 6`, so the budget is `B = ceil(9) + 2 = 11`.

**Ideal trajectory** (tool outputs abridged):

```
1 list_tables(pattern="%pos%")
  → pos_snap_2024q2, pos_snap_2024q3, pos_snap_2024q3_bak (comment: "DEPRECATED copy 2024-08-31"), stg_pos_load, ...
2 describe_table("pos_snap_2024q3")
  → fund_cd TEXT, secid INTEGER, qty REAL, avg_cst_usd REAL, asof_dt TEXT ("20240930")
3 list_tables(pattern="%px%")
  → px_eod_2023, px_eod_2024, px_eod_2024_vendor, ...
4 describe_table("px_eod_2024")
  → sec_id INTEGER, trd_dt TEXT ("2024-09-30"), cls_px REAL, adj_cls_px REAL, vol INTEGER
5 run_sql("SELECT p.secid, p.qty*(x.cls_px - p.avg_cst_usd) AS upl
           FROM pos_snap_2024q3 p JOIN px_eod_2024 x
             ON x.sec_id = p.secid AND x.trd_dt = '2024-09-30'
           WHERE p.fund_cd = 'GLB03' AND p.asof_dt = '20240930'
           ORDER BY upl LIMIT 3")
  → {"ok": false, "error": {"type": "ServiceUnavailable", ...}}      ← injected fault
6 run_sql(<same query>)
  → [[88121, -274200.0], [90417, -231900.0], [77310, -75350.0]]
7 run_sql("SELECT SEC_ID, TKR FROM SEC_MASTER WHERE SEC_ID IN (88121,90417,77310)
           AND VALID_FROM <= '2024-09-30' AND (VALID_TO IS NULL OR VALID_TO > '2024-09-30')")
  → [[88121, "NWCI"], [90417, "TLVQ"], [77310, "BRKX"]]
```

**Final answer:**

````
```json
{"answer_type": "entity_values",
 "answer": [{"entity": "NWCI", "value": -274200},
            {"entity": "TLVQ", "value": -231900},
            {"entity": "BRKX", "value": -75350}],
 "unit": "USD", "scale": "units"}
```
````

**Scoring:** 7 calls, with 1 injected fault, so the budget becomes 12. All
three entities are grounded and matched, and every value has `r = 0`. So
C = 1.0 and P = 0. **Total 1.0.** The fault was recovered at call 6.

**Variants:**

| Variant | Correctness | Penalties | Total |
|---|---|---|---|
| Ideal, but 17 calls (5 over budget) | 1.0 | p_budget = 0.10 | 0.90 |
| Ideal, plus 2 malformed calls (arguments not valid JSON) | 1.0 | p_invalid = 0.10 | 0.90 |
| Used `adj_cls_px`: NWCI and TLVQ values off by 1.8% and 2.4%, third entity wrong | soft F1: m = 0.406 + 0.268 = 0.674; P = R = 0.674/3 → C = 0.225 | 0 | 0.225 |
| Used the decoy `pos_snap_2024q3_bak`: 1 of 3 entities matched, value off by 30% | m = 0 → C = 0 | n/a | 0.0 |
| Answered `["NWCI","TLVQ","BRKX"]` after call 1 with no queries | entities never appeared in tool output → grounding gate → C = 0 | n/a | 0.0 |
| Got `SQLError: no such column: ticker` at the last call, then answered with IDs only from call 6 | C = 1.0 (internal IDs are accepted) | the error came after the last success: p_unrecovered = 0.05 | 0.95 |
| No final JSON before the 20-call limit | 0 | n/a | 0.0 |

### Example B: T2-01 (D2), scalar

**Question:** What was total revenue for ticker NWCI in fiscal 2023, in USD?

- **Traps:** revenue lives in `fs_fund_long` as tag
  `RevenueFromContractWithCustomerExcludingAssessedTax`, with `val_k` in
  **thousands**. It is keyed by zero-padded CIK text, and ticker → CIK comes
  from `SEC_MASTER`.
- **Hidden gold:** 12,388,000,000 USD.
- An answer of `{"value": 12388000, "unit": "USD", "scale": "thousands"}`
  canonicalises to 1.2388e10, so `r = 0` and the **total is 1.0**.
- An answer of `{"value": 12388000, "unit": "USD", "scale": "units"}` is off by
  a factor of 1000, so `scale_ok = 0` and the **total is 0**. This is the unit
  trap working as intended.
