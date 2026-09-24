# Task spec: `fin_numeric`

Single-turn numerical reasoning over excerpts from 10-K and 10-Q filings.

Status: design. None of this is implemented yet, and no results exist.

- Every number marked *target* or *initial value* is a guess that we will
  revisit after Milestone 1 (see [`../milestones.md`](../milestones.md)).
- **All numeric parameters in this spec are initial values unless stated
  otherwise.** That covers variation rates, residual and plausibility bounds,
  retry counts, weights, floors, and curve constants.

Contents:

1. [What the model sees and does](#1-what-the-model-sees-and-does)
2. [Data source and ground truth](#2-data-source-and-ground-truth)
3. [Rendering](#3-rendering)
4. [Question templates and difficulty levels](#4-question-templates-and-difficulty-levels)
5. [Contamination control: perturbation](#5-contamination-control-perturbation)
6. [Splits and target sizes](#6-splits-and-target-sizes)
7. [Output format](#7-output-format)
8. [Grader](#8-grader)
9. [Reward hacking: exploits and guards](#9-reward-hacking-exploits-and-guards)
10. [External evaluation: FinQA and TAT-QA](#10-external-evaluation-finqa-and-tat-qa)
11. [Worked examples](#11-worked-examples)

---

## 1. What the model sees and does

The model gets one prompt containing:

- a filing header: company name, form type, fiscal period, and the scale line
  ("in millions, except per-share data")
- one to three tables rendered as text, where every row has an ID
- zero to three short text paragraphs, where every paragraph has an ID
- one question asking for a single computed number

It writes free-form reasoning and then one JSON block: value, unit, scale,
formula, and cited rows. The grader returns a score in [0, 1] plus a
breakdown by component.

## 2. Data source and ground truth

**Source.** SEC EDGAR XBRL company facts, from the `companyfacts` API or its
bulk archive. Each fact carries the filing's accession number, form, fiscal
year and period, period start and end dates, and value. We group facts by
accession number, so every item is tied to exactly one filing. A 10-K item can
also use the comparative periods that the same 10-K reports.

**Concept resolution.** Companies tag the same economic concept differently.
Revenue, for example, may be `Revenues`,
`RevenueFromContractWithCustomerExcludingAssessedTax`, or `SalesRevenueNet` in
older filings. A hand-maintained file, `tasks/fin_numeric/concepts.yaml`, maps
each *canonical concept* (e.g. `revenue`, `cost_of_revenue`,
`operating_income`) to an ordered list of candidate us-gaap tags.

- A template can use a filing only if every concept it needs resolves to
  exactly one tag with one value per period.
- Ambiguous filings are skipped, and the skip is logged along with the
  template.

**Period selection.**

- Annual (FY) duration facts must span 350–380 days.
- 10-Q quarterly facts must span 80–100 days. Year-to-date facts must span
  roughly 180 or 270 days, and are labelled `6M` or `9M`.
- Instant facts, i.e. balance-sheet items, are keyed by period end date.

**Ground truth is computed, never looked up.**

- Every item stores a *reference program*: an arithmetic expression over cell
  IDs.
- Ground truth is that program evaluated on the **displayed** values, after
  perturbation and rounding.
- It is never an XBRL value the company reported for a derived metric. This
  way the answer can always be reproduced exactly from what the model sees.

**v1 universe.** Operating companies whose 10-K filings for FY2012–FY2025 use
us-gaap tags.

- Banks, insurers and REITs are excluded in v1 because their statements have a
  different structure. A later template pack can add them.
- *Alternative:* include them now with sector-specific templates. That gives
  broader coverage, but roughly doubles the work on concept mapping and
  identities before Milestone 1.

## 3. Rendering

**Decision: render tables synthetically from XBRL facts** rather than
extracting the filing's own HTML tables.

- *Alternative:* parse the actual HTML tables from the filing and align cells to
  XBRL facts.
- Real layouts are more realistic, but alignment is error-prone, perturbing
  the tables means editing HTML, and multi-level headers make row IDs
  fragile.
- We keep the real-layout option as a later eval split. See open question
  Q2 in [`../open_questions.md`](../open_questions.md).

To keep synthetic rendering from becoming a trivially regular format, it
varies the following:

| Dimension | Variation |
|---|---|
| Row labels | 2–6 label variants per concept, e.g. "Net sales", "Total net revenue", "Revenues" |
| Row order | shuffled within accounting-legal positions; totals always follow their components |
| Distractor rows | 2–8 rows not needed for the question, e.g. both subtotals and components |
| Column order | current period first (80%) or last (20%) |
| Table style | pipe markdown, fixed-width text, or "label ..... value" |
| Negatives | parentheses `(1,234)` or a minus sign, chosen per table |
| Scale line | "in millions" / "in thousands" / "$ in millions, except per share" |

**IDs.**

- Row IDs are stable within an item: `IS-01`… for the income statement,
  `BS-01`… for the balance sheet, `CF-01`… for cash flow, and `T-1`… for text
  paragraphs.
- Column keys are period labels such as `FY2024`, `FY2023` and `9M2024`.
- A cell is referenced as `IS-06[FY2024]`.

Text paragraphs come from about 30 sentence templates, e.g. restructuring
charges, share repurchases, dividends, segment notes, and one-time gains. They
are filled with the same (perturbed) values as the tables. Some paragraphs are
distractors, such as a charge from a prior year.

## 4. Question templates and difficulty levels

Difficulty follows the number and kind of operations the reference program
needs:

| Level | Definition | Reference program shape |
|---|---|---|
| L1 lookup | the answer is a cell or text value, verbatim | `x` |
| L2 single operation | one ratio, difference, or sum over one period | `a / b`, `a − b` |
| L3 multi-step | two or more operations, or two balance-sheet dates, within one filing | `(a − b) / b`, `a / ((b₁ + b₀) / 2)` |
| L4 cross-period / adjusted | combines several filings or periods (FY minus YTD, TTM) or adjusts with a text-disclosed item | `(a + t) / r − c / r'` |

Template set v1 (initial; about 29 templates):

| ID | Level | Question (one of 3–6 paraphrases) | Unit |
|---|---|---|---|
| L1-01 | L1 | What was {IS line} in {period}? | USD |
| L1-02 | L1 | What was {BS line} as of {date}? | USD |
| L1-03 | L1 | What was {CF line} in {period}? | USD |
| L1-04 | L1 | What was diluted EPS in {period}? | USD_per_share |
| L1-05 | L1 | According to the text, how much was {disclosed item}? | USD |
| L2-01 | L2 | Gross margin in {period} | percent |
| L2-02 | L2 | Operating margin in {period} | percent |
| L2-03 | L2 | Net margin in {period} | percent |
| L2-04 | L2 | Effective tax rate in {period} | percent |
| L2-05 | L2 | Current ratio as of {date} | ratio |
| L2-06 | L2 | Total debt to equity as of {date} | ratio |
| L2-07 | L2 | Free cash flow (CFO − capex) in {period} | USD |
| L2-08 | L2 | Absolute change in {line} from {p0} to {p1} | USD |
| L2-09 | L2 | {R&D or SG&A} as % of revenue in {period} | percent |
| L3-01 | L3 | Year-over-year growth in {line} | percent |
| L3-02 | L3 | Change in {margin} from {p0} to {p1}, in basis points | bps |
| L3-03 | L3 | Return on average equity for {period} | percent |
| L3-04 | L3 | Return on average assets for {period} | percent |
| L3-05 | L3 | Free cash flow margin in {period} | percent |
| L3-06 | L3 | Net debt as of {date} | USD |
| L3-07 | L3 | Interest coverage (operating income / interest expense) | ratio |
| L3-08 | L3 | EBITDA (operating income + D&A from the cash flow statement) | USD |
| L3-09 | L3 | Days sales outstanding on average receivables | days |
| L4-01 | L4 | Operating margin excluding {text-disclosed charge}, change vs prior year in bps | bps |
| L4-02 | L4 | Implied fourth-quarter {line} (10-K FY minus Q3 10-Q 9M YTD) | USD |
| L4-03 | L4 | Trailing-twelve-month {line} as of {Q} (YTD + prior FY − prior YTD) | USD |
| L4-04 | L4 | Three-year CAGR of {line} | percent |
| L4-05 | L4 | Diluted EPS excluding {charge}, net of tax at the disclosed rate | USD_per_share |
| L4-06 | L4 | YoY growth of TTM {line} | percent |

**L4 inputs.** L4-02, L4-03 and L4-06 put two filing excerpts in one prompt
(e.g. the 10-K and the Q3 10-Q), each with its own header, row IDs and column
keys.

**Generation filters.** An item is rejected if any of these hold:

- the ground truth is near zero: |y| < `min_abs` for its unit. The initial
  values are percent 0.5, bps 50, ratio 0.05, days 1, USD_per_share 0.05, and
  USD/shares 0.1% of the largest input. `min_abs` is separate from the grader
  floor in §8.2, which only sets the denominator of the relative error.
- an implied intermediate is implausible, e.g. an implied Q4 or TTM value with
  a sign opposite to its FY counterpart
- a denominator is ≤ 0 for a ratio template
- the resulting ratio is implausible (e.g. gross margin outside
  [−100%, 100%])

**Paraphrase hold-out.** Each template has 3–6 paraphrases. Two paraphrases
per template are used only in val and test. This measures dependence on
template wording.

## 5. Contamination control: perturbation

**Goal.** A model that memorised a company's public figures should gain
nothing. Ideally it should lose, because the memorised answer is
distinguishably wrong.

### 5.1 Statement tree

For each item we build a small accounting tree covering the rendered
statements. Totals are internal nodes and components are leaves. Examples of
the identities:

- `gross_profit = revenue − cost_of_revenue`
- `operating_income = gross_profit − Σ opex lines`
- `pretax_income = operating_income − interest_expense + other_income`
- `net_income = pretax_income − tax`
- `total_assets = Σ asset lines = total_liabilities + total_equity`
- `ending_cash = beginning_cash + CFO + CFI + CFF + fx_and_other`

Each total also gets a **residual leaf**, rendered as "Other, net", so the
identity holds exactly in display units on the *original* data. If a residual
is larger than 15% of its parent total, the item is rejected. That much
residual means the concept mapping is missing a major line.

### 5.2 Perturbation scheme

These are initial values, applied per item with a seed derived from `item_id`:

1. **Item scale** `s ~ LogUniform(0.5, 2.0)`, applied to every USD amount.
   This hides absolute company size.
2. **Row-level noise** `a_i ~ Uniform(−0.10, 0.10)` per leaf, which changes
   margins and mix.
3. **Period-level noise** `b_{i,t} ~ Uniform(−0.04, 0.04)` per leaf per period,
   which changes growth rates. It stays small so trends remain plausible.
   - For L4 items that span 10-Q and 10-K excerpts, noise is applied to
     **quarterly** leaves once.
   - YTD (6M/9M) and FY values are then *built* from the perturbed quarters and
     shared across the filings in the item, so FY − 9M = Q4 holds exactly.
4. Leaf value: `v'_{i,t} = v_{i,t} · s · (1 + a_i + b_{i,t})`, rounded to the
   display precision.
5. **Recompute every total bottom-up** from the rounded leaves, so totals add
   up exactly.
6. **Balance sheet.** Assets and liabilities leaves are perturbed. Total equity
   is set to `total_assets' − total_liabilities'`, and retained earnings acts as
   the plug inside equity.
   - This means the roll-forward RE_t = RE_{t−1} + NI − dividends is **not**
     preserved.
   - So items with two balance-sheet dates never include dividend or buyback
     text paragraphs, which could contradict it.
7. **Cash flow.** `fx_and_other` is the plug, so ending cash equals
   balance-sheet cash. Beginning cash for period t is ending cash for t−1.
8. **Cross-statement nodes** such as net income and cash are single nodes, so
   they print the same value everywhere.
9. **Shares.** `shares' = shares · (1 + c)` with `c ~ Uniform(−0.10, 0.10)`,
   drawn independently of `s`. Otherwise EPS would not move.
10. **Displayed EPS** is `round(net_income' / displayed_shares', 2)`, using
    the displayed (rounded) share count. Ground truth for EPS templates uses
    the displayed EPS.
    - A model that recomputes EPS from net income and shares can differ by up
      to $0.005 because of rounding.
    - The USD_per_share floor ($1.00, §8.2) keeps that inside `r0`.
11. **Text paragraphs** are filled from perturbed values. Disclosed items that
    are not in the tables get the same `s` plus their own row noise.

### 5.3 Constraints and checks

After perturbation, the item must pass all of these checks or it is re-drawn,
up to 5 times, and then dropped:

- **Sign preservation:** every non-plug leaf and total keeps its original
  sign. We reject rather than flip, so a "net loss" in the text stays a loss.
  - Plug leaves (`fx_and_other`, retained earnings, "Other, net") are exempt.
    Instead they are bounded: |plug| ≤ 15% of the parent total.
  - Rejection rates are logged by the original operating margin, so any bias
    against near-breakeven or loss-making companies is visible.
- **Components within parents:** a text-disclosed component, such as
  restructuring inside SG&A, must be ≤ the line that contains it.
- **Exact identities:** all identities hold exactly in integer display units.
  A failure here is a generator bug and fails the build, not just the item.
- **Separation:** measured with the grader's own error,
  `r_sep = |truth_original − truth| / max(|truth|, floor) ≥ 3%`.
  - At that distance, a memorised answer gets numeric ≤ 0.177 (§8.2). Because
    citations scale with numeric (§8.6), its total is also ≤ 0.177.
  - This is what makes leak detection possible.
- **Plausibility:** margins, ratios and growth rates stay inside per-template
  bounds.

Every item stores `truth_original` next to `truth` for leak diagnostics. The
model never sees it.

**Company names stay real** in perturbed items, so memorised knowledge is
actively misleading and leaks become measurable. The dataset card must say
clearly that perturbed figures are not the company's reported figures.

*Alternative:* anonymise names and years. That is safer if the data is
published, but it removes the leak signal. See Q3 in
[`../open_questions.md`](../open_questions.md).

### 5.4 Unperturbed control split

- `test_control` holds unperturbed twins of a 1,000-item subset of `test`: the
  same filings, templates, paraphrases and rendering seed, but with original
  values.
- **Memorisation gap** = `mean score(test_control) − mean score(matched test)`.
  A clearly positive gap means the model gets real figures right more often
  than perturbed ones, and the likeliest cause is recall.
- **Leak rate** = the fraction of perturbed test items with `leak_hit = 1`:
  - the answer is within `r0` of `truth_original`, with
    `r = |ŷ − truth_original| / max(|truth_original|, floor)`
  - and it is not within `r0` of `truth`
- **Confound:** perturbed tables can look slightly "off", e.g. unusual margins.
  To separate that effect from recall, we also report the gap on a
  **post-cutoff slice**: filings dated after the base model's training cutoff,
  where memorisation is impossible.

## 6. Splits and target sizes

All sizes are **targets**.

| Split | Size (target) | Companies | Perturbed | Use |
|---|---|---|---|---|
| `train` | 20,000 | ~70% of CIKs | yes | RFT, GRPO, DPO |
| `val` | 1,000 | ~10% of CIKs, disjoint | yes | feasibility check, checkpoint selection |
| `test` | 2,000 | ~20% of CIKs, disjoint | yes | headline metric; used only at milestone ends |
| `test_control` | 1,000 | subset of `test` | no | memorisation gap |
| `test_postcutoff` | 300–500 | CIKs not in `train`/`val`, filed after model cutoff | both variants | confound check |
| `ext_finqa`, `ext_tatqa` | 1,147 / TAT-QA dev arithmetic subset (§10) | n/a | no | transfer to real layouts |

- **Level mix** in every split: L1 15% / L2 30% / L3 30% / L4 25%.
- Splits are **CIK-disjoint**, so a company's figures never show up in more
  than one split.
- *Alternative:* a temporal split (train on older fiscal years, test on newer
  ones). That is closer to deployment, but it leaves the same companies on both
  sides. We use CIK-disjoint splits as primary and the post-cutoff slice as the
  temporal check.
- Each split is written as a JSONL file together with a manifest: generator git
  sha, seed, concept-map hash, and count per template. The split is identified
  by the sha256 of the file.

## 7. Output format

System instruction (abridged):

> Think step by step using only the figures provided. End your answer with
> exactly one fenced `json` block:
>
> ```json
> {"value": <number>, "unit": "<unit>", "scale": "<scale>",
>  "formula": "<expression over cell references>", "cited_rows": ["<id>", ...]}
> ```

| Field | Type | Allowed values |
|---|---|---|
| `value` | JSON number (finite) | no strings, no `%`, no thousands separators |
| `unit` | string | `USD`, `USD_per_share`, `shares`, `percent`, `bps`, `ratio`, `days` |
| `scale` | string | `units`, `thousands`, `millions`, `billions` |
| `formula` | string | arithmetic over `+ − * / ( )`, numeric literals, cell refs `ROW[COL]`, text refs `T-n` as literals |
| `cited_rows` | list of strings | row IDs (`IS-06`) or text IDs (`T-1`) |

**Parse rules:**

- Thinking content (`<think>…</think>`) is stripped first. An unclosed
  `<think>` makes the output unparseable.
- The parser takes the **last** fenced `json` block.
- The output is **unparseable** if any of these hold:
  - `value`, `unit` or `scale` is missing
  - the JSON is invalid
  - `value` is a boolean, a string, `NaN` or `Infinity`. The JSON parser
    rejects non-finite constants, and booleans are checked explicitly.
  - `unit` or `scale` is outside its enum
- If `formula` or `cited_rows` is missing, the output still parses. The
  citation component is then 0, and formula consistency is recorded as n/a.

## 8. Grader

`score(item, parsed) -> {total, components, info}`

### 8.1 Canonicalisation

1. **Unit families:** `{USD}`, `{USD_per_share}`, `{shares}`,
   `{percent, bps, ratio}`, `{days}`.
2. `unit_ok` = the declared unit is in the same family as the expected unit.
3. Canonical value `ŷ = value × scale_mult(scale) × conv(unit → expected_unit)`,
   where:
   - `scale_mult`: units 1, thousands 1e3, millions 1e6, billions 1e9.
     - It applies **only to `USD` and `shares`**.
     - For other units the `scale` field is ignored, and
       `scale_field_match = 0` is logged if it isn't `units`.
   - `conv` within the ratio family: ratio→percent ×100, percent→bps ×100,
     and so on
4. Ground truth `y` is stored in the expected unit at scale `units`.

### 8.2 Numeric accuracy (smooth decay)

Relative error with a unit-specific floor:

```
r = |ŷ − y| / max(|y|, floor)
```

The floor keeps small truths (e.g. 0.8% growth, which is above `min_abs`)
from turning tiny absolute slips into huge relative errors. Below the floor,
the error is effectively absolute. Floors are *initial values*:

| Expected unit | floor |
|---|---|
| percent | 10 (pp) |
| bps | 1000 |
| ratio | 0.1 |
| days | 10 |
| USD_per_share | 1.00 |
| USD, shares | 1% of the largest input magnitude in the reference program (cells and text literals such as T-1's 162), after scaling to units |

Decay curve (*initial values*: `r0 = 0.5%`, half-life `h = 1%`,
cutoff `r_max = 10%`):

```
numeric(r) = 1                          if r ≤ r0
           = 2 ^ ( −(r − r0) / h )      if r0 < r ≤ r_max
           = 0                          if r > r_max
```

| r | 0.5% | 1% | 1.2% | 2% | 3% | 5% | 10% |
|---|---|---|---|---|---|---|---|
| numeric | 1.000 | 0.707 | 0.616 | 0.354 | 0.177 | 0.044 | 0.001 → 0 |

- `r0` absorbs legitimate rounding. For example, a margin reported to two
  decimals is off by at most ~0.01%, well inside `r0`.
- The value at the `r_max` cutoff (0.0014) is small enough that dropping it to
  0 makes no practical difference.
- **Headline accuracy** is `acc@0.5%`, the share of items with `r ≤ r0`. It is
  reported next to the mean score so a smooth curve cannot hide a low exact
  rate.
- *Alternative:* a binary reward (1 if `r ≤ r0`). It is harder to game, but
  it gives no signal for near-misses. With 16 samples per prompt, that means
  more all-zero groups on L3/L4. Milestone 4 includes an ablation (binary vs
  smooth).

### 8.3 Sign and scale checks

- `sign_ok = sign(ŷ) == sign(y)`. Generation guarantees `|y| ≥ min_abs > 0`.
  Sign errors are common on losses, cash outflows, and negative growth.
- `scale_ok = |log10(|ŷ| / |y|)| < 0.5`, i.e. within a factor of about 3.16.
  If `ŷ = 0`, `scale_ok = 0`.
  - This catches thousands/millions confusion and percent-vs-ratio confusion
    (e.g. `0.394` percent for 39.4%), however the model labels it.
- **How much the gates matter.** Almost every sign or scale error already
  pushes `r` past `r_max`, so the two gates rarely change `total`. Their main
  job is failure-mode **diagnostics**: they separate "wrong scale" from "wrong
  arithmetic" in the breakdown. The gates are a backstop for the few cases
  where the floor would otherwise hide the error.
  - *Alternative:* partial credit, e.g. 0.2, for correct digits with the wrong
    scale or sign. That is a denser signal, but it rewards an error that is
    costly downstream. Not adopted.
- **Diagnostic** `scale_field_match`: the declared `scale` equals the source
  table's scale, for USD answers. It is logged but not rewarded, because
  answering in billions is legitimate when converted correctly.

### 8.4 Citations

- **Existence gate:** every ID in `cited_rows` must exist in the prompt. Any
  unknown ID sets `citation = 0` and `hallucinated_citation = 1`.
- **Otherwise**, `citation = max_j F1(dedupe(cited_rows), S_j)`.
  - `S_j` ranges over the template's acceptable citation sets. Templates define
    these as concepts, and the generator resolves them to **row IDs for each
    item**. For example, gross margin accepts `{revenue, gross_profit}` or
    `{revenue, cost_of_revenue}`, and a text-disclosed adjustment adds `T-n`.
  - An empty list scores 0.
- Using F1 rather than recall means citing every row does not pay.
- **Limitation:** citations are row-level, so a wrong-*period* error, like
  Example C's third variant, doesn't show up in the citation score. The
  numeric check catches it.
  - *Alternative:* cell-level citations (`IS-06[FY2023]`). They are more
    precise, but harder for the model to format, and there are more valid
    sets per template.

### 8.5 Formula consistency (diagnostic only in v1)

- `formula` is evaluated with a whitelist AST evaluator: numbers, cell refs,
  `+ − * /` and parentheses only.
- `formula_consistent = 1` if the result, after the same canonicalisation, is
  within `r0` of the declared value, measured as
  `|f − ŷ| / max(|ŷ|, floor)`.
- **Not rewarded in v1.** It is logged, and it is the main flag for spot
  checks: a correct value with an inconsistent formula means a lucky guess or
  recall.
- *Alternative:* reward it at weight 0.05–0.1. That makes the model show
  its work, but formula syntax errors would then cost reward on otherwise
  correct answers. See Q7.

### 8.6 Total

```
if not parsed or not unit_ok:
    total = 0
else:
    total = sign_ok · scale_ok · numeric · (0.9 + 0.1 · citation)
```

- **Numeric dominates.** Citations scale the score between 0.9× and 1.0× of
  `numeric`, so they only matter to the extent the number is right.
  - A well-cited answer 8% off earns about 0.005, not the 0.1 an additive
    citation term would pay.
  - This rewards grounding without rewarding well-cited wrong answers.
- *Alternative:* an additive `0.9·numeric + 0.1·citation`. That keeps a
  grounding signal on wrong answers, but pays for citation form on answers
  near the cutoff. Separately, 0.8/0.2 weights would put more pressure on
  grounding, at the cost of more room to trade accuracy for citation form.
- Components returned: `parsed`, `unit_ok`, `sign_ok`, `scale_ok`, `numeric`,
  `rel_error`, `citation`, `hallucinated_citation`, `scale_field_match`,
  `formula_consistent`, `leak_hit` (answer within `r0` of `truth_original`).

**Grader versioning.** The grader has a semantic version (`fin_numeric/grader@1.0.0`).

- Any change to weights, floors or the curve bumps the minor version.
- Scores from different grader versions are never compared in one results table.

## 9. Reward hacking: exploits and guards

| Likely exploit | Guard |
|---|---|
| Recite memorised public figures | Perturbation with a ≥3% separation constraint, `leak_hit` diagnostic, control-split gap |
| Cite every row to max out citations | F1 against acceptable sets; the existence gate zeroes invented IDs |
| Hedge with several JSON blocks or ranges | Only the last block counts; `value` must be one finite number |
| Unit juggling to land inside tolerance | Unit families are fixed, and conversion is applied before comparison |
| Aim for near-zero truths where the floor is generous | Generation rejects \|y\| < `min_abs`; below the floor, error is absolute, so tolerance is bounded |
| Right number, wrong reasoning (lucky or pattern-matched) | `formula_consistent` diagnostic, spot checks prioritise consistent=0 with numeric=1 |
| Overfit template wording | Paraphrase hold-out, label variants, distractor rows, external FinQA/TAT-QA eval |
| Positional heuristics (e.g. "revenue is always the first row") | Row order shuffled within legal positions; column order varies |
| Very long reasoning that happens to hit | Truncated output has no JSON and scores 0; the loss is length-bias-corrected (see design §7) |
| Answer placed in the thinking block only | Thinking is stripped before parsing |

**Spot checks.** The sampling, threshold and stop rule are defined once in
design §9: 20 random high scorers, plus detector hits, plus the biggest
risers. Training stops on more than 2 of 20 flagged or on any `exploit`.

For this task, the detector hits include outputs with
`formula_consistent = 0` and `numeric = 1`. The reviewer's checklist has four
items:
  - (a) the reasoning actually derives the value
  - (b) the cited rows are the ones used
  - (c) no figure in the reasoning matches an unperturbed original value
  - (d) there is no degenerate pattern (repeated text, boilerplate formula)

## 10. External evaluation: FinQA and TAT-QA

We use these public datasets for **evaluation only, never for training**:

- **FinQA** (Chen et al., 2021): numerical QA over S&P 500 earnings reports
  from 1999–2019, with annotated reasoning programs.
  - Released splits are train 6,251 / dev 883 / test 1,147. The private test
    set of 919 has no public answers.
  - We use `test`, all 1,147 items.
  - The code repo is MIT. The data terms still need confirmation (Q18).
- **TAT-QA** (Zhu et al., 2021): hybrid table-and-text QA from 182 financial
  reports.
  - Released splits are train 13,215 / dev 1,668 / test 1,663 (gold file).
  - We use the arithmetic-answer subset of `dev`, so results are comparable
    with prior published numbers.
  - The data is CC BY 4.0.

Split sizes above were counted from the datasets' GitHub releases. They get
re-counted when the external-eval manifest is built.

Why eval-only:

1. **They are our only check on real layouts and human-written questions.**
   Training on them would turn a transfer test into an in-distribution test.
2. **Their figures are unperturbed and have been public since 2021.** They are
   likely present in base-model pretraining data. Scores on them mix reasoning
   with recall, so they cannot measure training gains cleanly. We read them
   next to the control-split gap, not on their own.
3. **The output conventions differ.** They have no scale or citation fields,
   and the gold answers use their own rounding conventions. Training on them
   would teach those conventions instead of ours.
4. **Label quality is unaudited by us.** Before reporting, we audit a random
   100 items per dataset and exclude any we judge mislabelled. The exclusion
   list is versioned.

**Scoring adapter.** The adapter maps each gold answer to `(value, unit)`, and
scores only the numeric component on the same curve.

- If the gold answer has no explicit scale, `scale_ok` is dropped.
- Citations are not scored.
- The adapter is a separate grader version (`fin_numeric/ext@1.0.0`).

## 11. Worked examples

The company and every figure below are **illustrative and fictional**. They do
not describe any real issuer. Ground truth was computed from the displayed
values with the reference programs shown.

### Shared input excerpt (perturbed, as the model sees it)

```
Company: Northwind Components Inc.   Form: 10-K   Fiscal year ended Dec 31, 2024
Consolidated Statements of Operations (in millions, except per-share data)

| Row   | Line item                                   |  FY2024 |  FY2023 |
|-------|---------------------------------------------|---------|---------|
| IS-01 | Net sales                                   |  13,106 |  12,388 |
| IS-02 | Cost of sales                               |   7,941 |   7,702 |
| IS-03 | Gross profit                                |   5,165 |   4,686 |
| IS-04 | Selling, general and administrative         |   2,013 |   1,951 |
| IS-05 | Research and development                    |     648 |     611 |
| IS-06 | Operating income                            |   2,504 |   2,124 |
| IS-07 | Interest expense                            |     118 |     131 |
| IS-08 | Income before income taxes                  |   2,386 |   1,993 |
| IS-09 | Provision for income taxes                  |     501 |     426 |
| IS-10 | Net income                                  |   1,885 |   1,567 |
| IS-11 | Diluted earnings per share                  |    4.71 |    3.87 |
| IS-12 | Diluted weighted-average shares (millions)  |   400.3 |   404.9 |

T-1: Fiscal 2024 selling, general and administrative expenses include
restructuring charges of $162 million related to the consolidation of two
manufacturing sites. No restructuring charges were recorded in fiscal 2023.
```

### Example A: L2-01 gross margin

**Question:** What was the gross margin in fiscal 2024?

**Reference program:** `IS-03[FY2024] / IS-01[FY2024] × 100` = 5,165 / 13,106 × 100
= **39.4094%**.

**Ideal output:**

````
Gross margin is gross profit divided by net sales. FY2024 gross profit is
5,165 and net sales are 13,106 (both in millions), so 5,165 / 13,106 = 0.39409,
or 39.41%.

```json
{"value": 39.41, "unit": "percent", "scale": "units",
 "formula": "IS-03[FY2024] / IS-01[FY2024] * 100", "cited_rows": ["IS-03", "IS-01"]}
```
````

Score: `r = |39.41 − 39.4094| / max(39.4094, 10) = 0.0015%` → numeric 1.0.
Citation F1 = 1.0 against the set `{IS-01, IS-03}`. **Total 1.0.**

| Variant output | What happens | Total |
|---|---|---|
| `"value": 0.3941, "unit": "ratio"` | ratio→percent ×100 = 39.41; all checks pass | 1.0 |
| `"value": 0.3941, "unit": "percent"` | ŷ = 0.3941 vs 39.41, factor 100 → `scale_ok = 0` | 0.0 |
| `"value": 39.41`, cited `["IS-01","IS-02"]` | the `{revenue, cost_of_revenue}` set is acceptable → citation 1.0 | 1.0 |
| `"value": 39.41`, cited `["IS-01","IS-03","IS-14"]` | IS-14 does not exist → citation 0 | 0.9 |
| `"value": 39.88` | r = 1.194% → numeric 2^(−0.694) = 0.618; citation 1.0 | 0.618 · (0.9 + 0.1) = 0.618 |
| no JSON block | unparseable | 0.0 |

### Example B: L3-01 EPS growth

**Question:** By what percentage did diluted EPS grow from fiscal 2023 to
fiscal 2024?

**Reference program:** `(IS-11[FY2024] / IS-11[FY2023] − 1) × 100`
= (4.71 / 3.87 − 1) × 100 = **21.7054%**.

A model that recomputes EPS from net income and shares gets
(1,885/400.3) / (1,567/404.9) − 1 = 21.676%. That is `r = 0.14%`, inside `r0`,
so it scores numeric 1.0. Citing `{IS-10, IS-12}` is listed as an acceptable
set for this template.

| Output value | r | numeric | Total |
|---|---|---|---|
| 21.71 | 0.02% | 1.0 | 1.0 |
| 21.67 | 0.16% | 1.0 | 1.0 |
| 20.29 (used net income growth; cited only IS-10, so F1 = 0.67 vs `{IS-10, IS-12}`) | 6.52% | 0.015 | 0.015 · (0.9 + 0.067) = 0.015 |
| −21.71 | sign wrong | n/a | 0.0 |

### Example C: L4-01 adjusted operating margin change

**Question:** Excluding restructuring charges, by how many basis points did
operating margin change from fiscal 2023 to fiscal 2024?

**Reference program:**
`((IS-06[FY2024] + 162) / IS-01[FY2024] − IS-06[FY2023] / IS-01[FY2023]) × 10000`

Here 162 comes from T-1. Adjusted FY2024 margin = 2,666 / 13,106 = 20.3418%.
FY2023 margin = 2,124 / 12,388 = 17.1456%. Change = **319.62 bps**.

**Ideal output:**

````
Restructuring charges of $162M sit inside FY2024 SG&A (T-1), so adjusted FY2024
operating income is 2,504 + 162 = 2,666. Adjusted margin = 2,666 / 13,106 =
20.342%. FY2023 had no restructuring, so its margin is 2,124 / 12,388 = 17.146%.
Change = 3.196 percentage points = 319.6 bps.

```json
{"value": 319.6, "unit": "bps", "scale": "units",
 "formula": "((IS-06[FY2024] + 162) / IS-01[FY2024] - IS-06[FY2023] / IS-01[FY2023]) * 10000",
 "cited_rows": ["IS-06", "IS-01", "T-1"]}
```
````

**Score: total 1.0.** Numeric error `r = 0.02 / 1000 ≈ 0` (the floor is 1000
bps because |y| < 1000), and citation F1 = 1.0.

| Variant | What happens | Total |
|---|---|---|
| `"value": 3.196, "unit": "percent"` | percent→bps ×100 = 319.6 | 1.0 |
| ignored T-1: 196.0 bps | r = 123.6 / 1000 = 12.4% > r_max → numeric 0; citation gated off | 0.0 |
| added T-1 to FY2023 as well (wrong period) | 2,286/12,388 = 18.453% → 188.8 bps; r = 13.1% | 0.0 |
| `"value": 320` | r = 0.04% | 1.0 |
