## Review Summary
- **Round**: 1
- **Theme**: Broad sweep
- **Mode**: sequential
- **Model**: claude-opus-5
- **Artifact**: reviews/ai-functions/task-unknown-attempt-1-review-1-claude-opus-5.md
- **Issues Found**: 4
- **Verdict**: ISSUES_FOUND

## Evidence Checklist
- [x] Enumerated the staged change set: `git --no-pager diff --staged --stat` shows exactly six new files, 2812 insertions, all under `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/`.
- [x] Parsed every notebook's cell sources (not raw JSON): 17/17/16/16/16/16 cells; all `execution_count` are `null`, all `outputs` arrays are empty, zero blank-source cells, all cells carry unique `id` values, `nbformat` 4.5 in all six.
- [x] Verified model routing per requirement: every notebook's cell 2 defines `EXECUTOR_OPTIONS = {"deploymentName": "gpt-5-mini", "reasoningEffort": "low"}` and `JUDGE_OPTIONS = {"deploymentName": "gpt-5.1", "reasoningEffort": "medium"}`; traced each call site — `ai.analyze_sentiment`/`ai.classify`/`ai.extract`/`ai.fix_grammar`/`ai.summarize`/`ai.translate` and all custom-prompt refinements use `**EXECUTOR_OPTIONS`; every `add_judge_metric`/eval `ai.generate_response` uses `**JUDGE_OPTIONS`. No crossover found.
- [x] Confirmed prohibited executable parameters absent: case-insensitive scan for `temperature`, `top_p`, `topP`, `seed` across all six files returned zero matches (markdown included).
- [x] Confirmed API type is never set explicitly, and verified against the public package that `responses` is the default API type — so omission is correct.
- [x] Confirmed verbosity handling: `verbosity` appears only in `AIFunctions-PySpark-eval-summarize.ipynb`, 4 occurrences, all in markdown (cells 1, 12, 15) including one non-executed fenced snippet in cell 15. No executed cell sets it; shared verbosity is never configured. Verified `verbosity` is a valid PySpark option key, so the optional snippet is accurate guidance.
- [x] Verified all public API signatures used are correct: `generate_response(prompt, is_prompt_template, output_col, error_col, response_format, col_types, **options)` accepts a Pydantic class for `response_format`; `summarize(...)` accepts `instructions`; `extract(labels=[str|ExtractLabel], input_col, raw_col, ...)` names output columns after the labels (so `select("text", "person", "organization", "location")` and `advanced_fields + ["role"]` resolve); `ExtractLabel(label, *, description, max_items, type, properties)` matches the keyword usage in `extract` cell 13; `translate(to_lang=..., input_col=...)`, `classify(labels=..., input_col=...)`, `analyze_sentiment(input_col=...)` all match.
- [x] Verified `ai.analyze_sentiment` default labels are exactly `positive, negative, neutral, mixed` (lowercase), matching the judge's `Literal[...]` in `analyze_sentiment` cell 8 — so `accuracy_score(y_true, y_pred)` in cell 11 compares a consistent label space and cannot be silently distorted by case/label drift.
- [x] Verified `import synapse.ml.spark.aifunc as aifunc` is **not** a dead import: importing the package is what registers the DataFrame `.ai` accessor, and it is additionally dereferenced as `aifunc.ExtractLabel` in `extract` cell 13. All other imports are used (`plt` in every chart cell, `pd` in every summary frame, `F` throughout, `BaseModel`/`Field` in every schema, `Literal` in the two classification notebooks, sklearn symbols in both cells 11 and 15).
- [x] Verified prompt-template placeholder resolution for every judge and custom prompt: `{text}`, `{sentiment}`, `{category}`, `{_categories}`, `{_extracted_summary}`, `{corrected}`, `{custom_corrected}`, `{article}`, `{summary}`, `{concise_summary}`, `{_target_lang}`, `{translation}`, `{custom_translation}` each correspond to a column present on the frame at call time. Confirmed the `.replace("{corrected}", ...)` / `.replace("{translation}", ...)` / `.replace("{summary}", ...)` rewrites in `fix_grammar` cell 13, `translate` cell 13, and `summarize` cell 13 hit only the placeholder token (the XML tags `<corrected_text>`, `<translation>`, `<summary>` contain no braces), so no prompt is silently mangled.
- [x] Verified no judge-label leakage into executor prompts: prompt templates substitute only the referenced columns, so running the custom executor over `evaluated_df` (which carries `expected_sentiment` / `expected_category` / `_eval_response`) in `analyze_sentiment` cell 14 and `classify` cell 14 does not feed the judge's answer to the model under test. The refinement comparison is methodologically sound.
- [x] Verified Spark-native execution and pandas boundary: all AI calls, JSON parsing (`F.get_json_object`), score casting, and joins stay in Spark; `toPandas()` is only applied to `select`-narrowed result sets of 5–8 rows (`analyze_sentiment` c11/c15, `classify` c11/c15, `extract` c10/c14, `fix_grammar` c10/c14, `summarize` c14, `translate` c10/c14).
- [x] Verified `.cache()` + `.count()` materialization is applied to every reused AI output via the `materialize` helper, and that `materialize` returns the same DataFrame object it was given (`DataFrame.cache()` returns `self`), so the `.ai.stats` accessor binding on `sentiment_df`/`classified_df`/`baseline_df`/`corrected_df`/`summary_df`/`translation_df` survives and `display(x.ai.stats)` resolves in all six notebooks.
- [x] Verified error-column semantics: on success the AI error column is `NULL`, so `F.coalesce(F.length(F.trim(col.cast("string"))) > 0, F.lit(False))` cannot false-positive on clean rows; `display(failed_rows.select("sample_id", *error_columns))` is safe because `sample_id` exists on every frame passed to `materialize`.
- [x] Reproduced the error-column naming collision deterministically for the `fix_grammar` chain, producing `text_fix_grammar_error`, `generate_response_error`, `generate_response_error_1`, `generate_response_error_2` — the helper's `endswith("_error")` filter detects only the first two (see Issue 1).
- [x] Verified every documentation link resolves: all six `learn.microsoft.com/fabric/data-science/ai-functions/pyspark/{analyze-sentiment,classify,extract,fix-grammar,summarize,translate}` targets plus the overview page returned HTTP 200 with no redirect.
- [x] Diffed each PySpark notebook against its pandas counterpart for dataset, judge-criteria, and refinement-path preservation; sample records match one-for-one for all six functions, judge rubrics are semantically equivalent, and section structure maps 1:1 except for the two omissions recorded in Issue 4.
- [x] Verified metric arithmetic consistency: baseline vs. refinement "Overall" rows use the same `mean(axis=1).mean()` construction on both sides in `extract` c14, `fix_grammar` c14, and `translate` c14; `dropna` guards and the `raise ValueError` empty-input guard in `analyze_sentiment` c11 / `classify` c11 are correct.

## Issues

### Issue 1: `materialize()` silently fails to detect AI errors from every judge call after the first
**Severity:** Medium
**Files / cells:**
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-analyze_sentiment.ipynb` — cell 2 (`materialize`), cell 14
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb` — cell 2, cell 14
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-extract.ipynb` — cell 2, cell 8, cell 13
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-fix_grammar.ipynb` — cell 2, cell 8, cell 13
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-summarize.ipynb` — cell 2, cell 8
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-translate.ipynb` — cell 2, cell 8, cell 13

**Problem:** The shared helper detects failures with
```python
error_columns = [name for name in cached.columns if name.endswith("_error")]
```
None of the notebooks pass an explicit `error_col` to `ai.generate_response`, so each call auto-generates a default error column and de-duplicates the name against the columns already on the frame. Because the judge metrics are applied as a *chain* on one frame, the second and later calls receive numerically suffixed names, which do not end in `_error` and are therefore never inspected.

**Evidence:** Deterministic reproduction of the `fix_grammar` cell 8 chain (`corrected_df` → coherence → consistency → grammar) using the package's documented auto-naming/uniquification behavior:

```
after ai.fix_grammar: ['sample_id', 'text', 'corrected', 'text_fix_grammar_error']
  judge[coherence]   errorCol -> generate_response_error
  judge[consistency] errorCol -> generate_response_error_1
  judge[grammar]     errorCol -> generate_response_error_2

materialize() detects: ['text_fix_grammar_error', 'generate_response_error']
MISSED               : ['generate_response_error_1', 'generate_response_error_2']
```
The same pattern applies to `summarize` cell 8 (5 chained metrics → `_1` through `_4` missed), `translate` cell 8 and cell 13, `extract` cell 8 and cell 13, and to the refinement executor calls in `analyze_sentiment` cell 14 and `classify` cell 14 (`evaluated_df` already carries `generate_response_error`, so the custom call's error column becomes `generate_response_error_1` and is missed). Confirmed independently that on success the error column is `NULL` and on failure it is non-null, so these columns are the intended failure signal.

**Risk:** Judge or executor failures are silently swallowed. Failed rows produce a null response column, which flows into `get_json_object(...) → null` scores. In `extract` / `fix_grammar` / `summarize` / `translate` those rows vanish from `mean()` with only the "Scored rows" column hinting at it; in `analyze_sentiment` cell 15 and `classify` cell 15 the `valid = expected.notna() & predictions.notna()` mask silently drops them with **no message at all**, so the Baseline and Custom columns of the comparison table can be computed over different row subsets and presented as a directly comparable Delta. The helper's entire purpose — surfacing AI Function errors before metrics are trusted — is defeated for the majority of calls in every notebook.

**Suggested fix:** Either pass an explicit, deterministic `error_col=f"_{score_col}_error"` on each `ai.generate_response` call (in `add_judge_metric` and the custom-refinement cells) and inspect those known names, or widen the detection predicate to also match a numeric suffix (e.g. `re.search(r"_error(_\d+)?$", name)`).

---

### Issue 2: `classify` hardcodes the category set in three places, so customizing `CATEGORIES` breaks the evaluation
**Severity:** Medium
**File:** `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb`
**Cells:** 4 (`CATEGORIES`), 8 (`Category = Literal[...]`, `ClassifyEval.expected_category`), 14 (`CustomClassifyResult.category`, `CUSTOM_PROMPT` category descriptions)

**Problem:** The category set is declared three times and only one of them is the variable the notebook tells users to edit:
- cell 4: `CATEGORIES = ['technical_support', 'billing', 'feedback', 'general_inquiry']` — drives `ai.classify(labels=CATEGORIES, ...)` and the `_categories` prompt column.
- cell 8: `Category = Literal["technical_support", "billing", "feedback", "general_inquiry"]`, used as the type of `ClassifyEval.expected_category` — a structured-output schema that **constrains** the judge's answer.
- cell 14: `CUSTOM_PROMPT` restates the same four categories in prose, and `CustomClassifyResult.category` reuses the same stale `Category`.

**Evidence:** Cell 0 of the same notebook instructs: "**Customize it** - Replace the sample data and adapt the judge criteria to your use case." A user who follows that instruction and edits only `CATEGORIES` gets `ai.classify` predicting the new labels while `expected_category` remains schema-constrained to the four original labels. `y_true` and `y_pred` then occupy disjoint label spaces, so `accuracy_score` in cell 11 returns ~0 and `precision_score`/`recall_score`/`f1_score` with `zero_division=0` return 0 — the notebook reports a total quality failure that is purely an artifact of the stale schema. The pandas counterpart does not have this failure mode: `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-pandas-eval-classify.ipynb` cell 10 declares `expected_category: str = Field(...)` (free-form) and supplies the valid set at runtime through the `{_categories}` template variable, so editing `CATEGORIES` propagates correctly. This is a behavioral regression introduced by the PySpark rewrite.

**Risk:** The primary advertised customization path silently produces a fully incorrect evaluation, with no error and no warning — the most damaging failure mode for a template notebook, because the numbers look real.

**Suggested fix:** Derive the judge schema from the single source of truth — build the constrained field from `CATEGORIES` at runtime (e.g. `Literal[tuple(CATEGORIES)]`, a dynamically built `Enum`, or `Field(json_schema_extra={"enum": CATEGORIES})`) — and generate the `CUSTOM_PROMPT` category block from `CATEGORIES` rather than restating it in prose.

---

### Issue 3: The "Interpreting Results" action table in the two classification notebooks uses a 1–5 scale that neither notebook produces
**Severity:** Medium
**Files / cells:**
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-analyze_sentiment.ipynb` — cell 16
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb` — cell 16

**Problem:** Both notebooks close with a decision table keyed on a 1–5 "Average score":
```
| Average score | Suggested action |
| **4.5-5.0**   | Strong candidate for production validation |
| **4.0-4.4**   | Good; inspect the lowest-scoring samples |
| **3.5-3.9**   | Acceptable for iteration; refine data, prompts, or labels |
| **Below 3.5** | Investigate before broader use |
```
Neither notebook computes any 1–5 score. Their only outputs are accuracy, macro precision, macro recall, and macro F1 — all bounded in `[0, 1]` — plus a per-class `classification_report` table. Correspondingly, the charts in cell 11 use `ax.set_ylim(0, 1)` with reference lines at `y=0.9` (analyze_sentiment) and `y=0.8` (classify).

**Evidence:** Every threshold in the table is unreachable: the best achievable value in these notebooks is `1.0`, which falls into the "Below 3.5 → Investigate before broader use" band. The table is a verbatim copy of the guidance in `extract` cell 15, `fix_grammar` cell 15, `summarize` cell 15, and `translate` cell 15, where the judge genuinely emits `score: int` values of 1–5 (`MetricEval` in cell 2) and the table is correct. The pandas counterparts of both classification notebooks carry a scale-appropriate guide instead — `AIFunctions-pandas-eval-analyze_sentiment.ipynb` cell 22 and `AIFunctions-pandas-eval-classify.ipynb` cell 20 both use `80%+ Excellent / 70%-80% Good / <70% Needs Work` — so this is a copy-paste regression, not an inherited defect.

**Risk:** A reader who follows the shipped release-readiness guidance will conclude that a perfectly-scoring classifier must be "investigated before broader use," directly inverting the ship/no-ship decision the section exists to support.

**Suggested fix:** In these two notebooks, replace the 1–5 band table with thresholds on the 0–1 metrics actually computed (mirroring the pandas percentage bands, and consistent with the `0.9` / `0.8` reference lines already drawn in cell 11).

---

### Issue 4: Two optional refinement paths from the pandas notebooks are not carried over
**Severity:** Low
**Files:**
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-summarize.ipynb` — no counterpart to pandas section 6
- `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-analyze_sentiment.ipynb` — no counterpart to pandas section 3.1

**Problem:** The requirement is to preserve the optional refinement paths of the pandas notebooks. Four of six notebooks do so exactly (`classify` §6 → PySpark §6; `extract` §3.1 + §6 → PySpark §6, which both runs the richer `ExtractLabel` extraction and compares it; `fix_grammar` §6 → PySpark §6; `translate` §6 → PySpark §6). Two do not:
- `AIFunctions-pandas-eval-summarize.ipynb` has **two** optional sections — §6 "(Optional) Summarize Multi-Column Data" (cell 18, the `df_tickets` dataset with `ticket_id`/`customer`/`issue` columns) and §7 "(Optional) Refine with the `instructions` Parameter". The PySpark version preserves only the `instructions` path (§6, cells 12–14). The multi-column path and its `df_tickets` sample dataset are reduced to a single prose bullet in cell 15: "For multi-column rows, omit `input_col` to summarize the full row."
- `AIFunctions-pandas-eval-analyze_sentiment.ipynb` has §3.1 "(Optional) Custom Sentiment Labels" (cell 9, demonstrating emotion-based, intensity-based, and customer-service label sets) plus §5.1 "(Optional) Refinement: Baseline vs Explainable Custom Sentiment". The PySpark version preserves only the latter (§6, cells 13–15); the custom-labels path becomes a prose bullet in cell 16: "Use domain-specific sentiment labels when the default four labels are too broad."

**Evidence:** Section-heading inventory of all pandas notebooks vs. the staged PySpark notebooks (see checklist item on the pandas/PySpark structural diff). Both omitted sections are executable, dataset-backed demonstrations in the pandas originals; the `labels=` parameter needed for the sentiment path and the `input_col`-optional behavior needed for the multi-column path are both available on the PySpark surface, so neither omission is forced by an API gap.

**Risk:** Requirement non-conformance and reduced parity between the two notebook families; readers migrating from the pandas notebooks lose two runnable refinement examples, including the only demonstration of multi-column summarization.

**Suggested fix:** Add a Spark-native equivalent of each omitted optional section — a small multi-column `df_tickets` frame summarized with `input_col` omitted in the summarize notebook, and a custom-`labels` `ai.analyze_sentiment` variant in the sentiment notebook — or, if the omissions are deliberate scope reductions, confirm that with the requirement owner and record the decision.

---

## Resolution Log

### Issue 1
- **Status**: Fixed
- **What changed**: Every AI Function call now supplies a deterministic `error_col`. The shared judge helper uses `error_col=f"_{score_col}_error"`, while executor, judge, custom, concise, advanced extraction, and multi-column summary calls use distinct named error columns.
- **Why**: Explicit names prevent Spark's auto-deduplication suffixes from bypassing the error-surfacing helper.
- **How verified**: AST validation confirms every `analyze_sentiment`, `classify`, `extract`, `fix_grammar`, `summarize`, `translate`, and `generate_response` call has an `error_col`. All notebooks also passed the local Spark mock execution suite.

### Issue 2
- **Status**: Fixed
- **What changed**: `CATEGORY_GUIDANCE` is now the single classification source of truth. `CATEGORIES` is derived from its keys, both prompt columns are generated from it, and judge/custom schemas use `str` fields instead of a duplicated hardcoded `Literal`.
- **Why**: Editing the category guidance now updates the executor and both prompts without leaving a stale structured-output constraint.
- **How verified**: The generated classification notebook contains no `Literal[...]` category schema or hardcoded category prose outside `CATEGORY_GUIDANCE`; its full code path passed the Spark mock execution suite.

### Issue 3
- **Status**: Fixed
- **What changed**: The sentiment and classification notebooks now use a 0-1 metric guide with 0.80 and 0.70 thresholds. The 1-5 guide remains only in notebooks that produce 1-5 judge scores.
- **Why**: Guidance now matches the actual accuracy, precision, recall, and F1 scale.
- **How verified**: Targeted content assertions found the 0.80-1.00 and below-0.70 bands in both classification notebooks and no 4.5-5.0 bands.

### Issue 4
- **Status**: Fixed
- **What changed**: The sentiment notebook adds runnable emotion, intensity, and customer-service label schemes. The summarization notebook adds the pandas ticket dataset as a Spark DataFrame and demonstrates full-row summarization by omitting `input_col`.
- **Why**: These restore the two executable parity paths that had been reduced to prose.
- **How verified**: Notebook section checks confirm both new optional sections, and the complete six-notebook Spark mock execution suite passes.

## Round 1 Re-review

## Review Summary
- **Round**: 1 (re-run)
- **Theme**: Broad sweep
- **Mode**: sequential
- **Model**: claude-opus-5
- **Artifact**: `reviews/ai-functions/task-unknown-attempt-1-review-1-claude-opus-5.md`
- **Issues Found**: 1
- **Verdict**: ISSUES_FOUND (all 4 prior findings resolved; 1 new residual issue introduced by the Issue 2 fix)

## Prior Findings — Resolution Verification

### Issue 1 — Deterministic `error_col` + full error surfacing → **RESOLVED**
- AST scan of all 21 AI-function call sites across the six notebooks: every `analyze_sentiment` / `classify` / `extract` / `fix_grammar` / `summarize` / `translate` / `generate_response` call passes an explicit `error_col`. Zero calls rely on auto-naming, so the library's `_1`/`_2` de-duplication path is never entered.
- Every explicit name ends in `_error` (`executor_error`, `advanced_executor_error`, `concise_executor_error`, `ticket_summary_error`, `_eval_error`, `_custom_error`, `{output_col}_error`, and the helper's `_{column_prefix}{metric}_error`), so `materialize`'s `endswith("_error")` predicate matches all of them. I traced every DataFrame lineage: no explicit `error_col` collides with a pre-existing column, so no silent renaming occurs.
- **Executed verification**: I ran all six notebooks end-to-end against a local Spark 3.4.1 session with a mocked `.ai` accessor, injecting a hard failure (null output + non-null error string) on one row of *every* AI call. All 26 `materialize` sites printed `N row(s) contain AI Function errors` and displayed the offending rows; 0 exceptions across all six notebooks. The `id_columns` fallback correctly picks `ticket_id` for the new multi-column ticket frame.

### Issue 2 — Single source of truth for categories → **RESOLVED** (see new Issue A for the residual)
- `CATEGORY_GUIDANCE` (cell 4) is now the only place categories are declared. `CATEGORIES = list(CATEGORY_GUIDANCE)` drives `ai.classify(labels=...)`, `_categories` (judge prompt), and `_category_guidance` (custom prompt). Grep confirms no `Literal[...]` and no hardcoded category prose survive anywhere in the notebook. Editing `CATEGORY_GUIDANCE` alone now propagates to the executor and both prompts.

### Issue 3 — Metric scale in the classification notebooks → **RESOLVED**
- `analyze_sentiment` cell 18 and `classify` cell 16 now use `0.80-1.00 / 0.70-0.79 / Below 0.70`, matching the `[0,1]` accuracy/precision/recall/F1 actually computed and the `ax.set_ylim(0, 1)` + `y=0.9` / `y=0.8` reference lines. The 1–5 band table survives only in `extract` / `fix_grammar` / `summarize` / `translate`, where `MetricEval.score: int` genuinely emits 1–5. No 4.5-5.0 bands remain in the two classification notebooks.

### Issue 4 — Missing parity paths → **RESOLVED**
- `analyze_sentiment` §3.1 (cells 7–8): runnable `CUSTOM_LABEL_SCHEMES` with the same emotion / intensity / urgency label sets as pandas cell 9, executed via chained `ai.analyze_sentiment(labels=..., output_col=..., error_col=...)`. Verified `labels` is a real public parameter and that the judge's `Literal["positive","negative","neutral","mixed"]` still matches the library's documented default label set exactly, so the custom-label demo does not contaminate the judged baseline.
- `summarize` §6 (cells 12–13): the pandas `df_tickets` dataset is carried over in full — all five columns (`ticket_id`, `customer`, `issue`, `priority`, `resolution`) and all three tickets — summarized with `input_col` omitted. Verified against the public signature that `input_col` is optional and omission summarizes the whole row.

## Requirements Re-verification
- **Model routing**: all six notebooks define `EXECUTOR_OPTIONS = {gpt-5-mini, low}` / `JUDGE_OPTIONS = {gpt-5.1, medium}`; every executor/custom/refinement call uses `**EXECUTOR_OPTIONS`, every judge call uses `**JUDGE_OPTIONS`. No crossover.
- **Prohibited parameters**: case-insensitive scan for `temperature`, `top_p`/`topP`, `seed`, `api_type`/`apiType` across all cells (code + markdown) returns zero matches.
- **Verbosity**: appears only in `summarize` markdown (cells 1, 14, 17) — never in an executed cell, shared default never configured. Confirmed `verbosity` is a valid PySpark option key, so the optional snippet is accurate.
- **Spark-native**: all AI calls, JSON parsing, casts, and the cell-16 join stay in Spark; `toPandas()` appears exactly twice per notebook, always on a `select`-narrowed 3–8 row frame.
- **Prompt templates**: I re-tokenized all 28 templated prompts (including the post-`.replace()` variants) using the template grammar the library actually applies, then evaluated each token against the live frame. All 28 parse cleanly, every token resolves to an existing **string** column, and no prompt contains a stray brace. Confirmed only referenced columns are sent, so running the custom executor over `evaluated_df` does not leak `expected_sentiment` / `expected_category` to the model under test.
- **Clean notebooks**: 19/17/16/16/18/16 cells; zero non-empty `outputs`, zero non-null `execution_count`, no cell metadata, no widget state, unique cell ids, `nbformat.validate` passes on all six.
- **Links**: all six `learn.microsoft.com/.../pyspark/*` targets plus the overview page return HTTP 200.

## Issues

### Issue A: Removing the `Literal` category constraint left the judge and custom classifier free-form and unnormalized, so label drift silently zeroes the metrics
**File:** `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb`
**Cells:** 8 (`ClassifyEval.expected_category`), 11 (metrics), 14 (`CustomClassifyResult.category`), 15 (comparison)
**Severity:** Medium

**Problem:** The Issue 2 fix eliminated the duplicated `Literal[...]` by replacing it with a plain `str` in *both* schemas:
- `expected_category: str = Field(description="Best category from the provided list")` — this is `y_true` in cell 11.
- `CustomClassifyResult.category: str = Field(description="Best category from the provided list")` — this is the `Custom` column in cell 15.

That removed the structured-output enum constraint that had been the only guarantee those values landed in the exact `snake_case` label space produced by `ai.classify`. Nothing replaced it: grep across the notebook's code shows the sole normalization is `F.trim(...)` on `custom_category`, and `expected_category` is not normalized at all (`lower`, `replace`, `strip` — zero occurrences).

**Evidence:**
1. The pandas notebook this was ported from does *not* rely on the model returning exact labels — `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-pandas-eval-classify.ipynb` cell 18 normalizes explicitly: `.astype(str).str.strip().str.lower().str.replace(" ", "_", regex=False)`. The PySpark version kept only the `strip` half of that chain. The presence of that normalization upstream is direct evidence the maintainers observed this drift in practice.
2. The impact is total, not marginal. Reproduced with sklearn on the notebook's own label space:
   ```
   y_pred (ai.classify)  = [technical_support, billing, feedback, general_inquiry, technical_support]
   y_true exact          -> accuracy 1.0   macro F1 1.0
   y_true "Technical Support"/"Billing"/... -> accuracy 0.0   macro F1 0.0
   ```
   A single formatting convention difference in the judge's free-form string flips the notebook from a perfect score to a total failure, with no error and no warning — the same silent-wrong-numbers failure mode Issue 2 was raised for, reached through a different door. Cell 12's "incorrect predictions" table would list every row, and the 0.80/0.70 guidance table would tell the reader to refine their labels.
3. The prior review's own suggested fix listed three remedies — `Literal[tuple(CATEGORIES)]`, a dynamically built `Enum`, or `Field(json_schema_extra={"enum": CATEGORIES})` — all of which satisfy the single-source-of-truth requirement *and* keep the output constrained. The implemented fix chose none of them.

**Suggested fix:** Derive the constraint from `CATEGORIES` at runtime rather than dropping it — e.g. build the field type from `Literal[tuple(CATEGORIES)]` / a generated `Enum`, or attach `json_schema_extra={"enum": CATEGORIES}` — for both `ClassifyEval.expected_category` and `CustomClassifyResult.category`. Failing that, apply the pandas normalization (`trim` + `lower` + space→underscore) to `expected_category` as well as `custom_category` so both sides share the label space of `ai.classify`.

---

No other issues found. The remaining duplications I checked (`ENTITY_LABELS` vs. the hardcoded `select` in `extract` cell 10; `EVAL_METRICS.keys()` vs. the `METRICS` lists in `fix_grammar`/`summarize`/`translate` cell 10) fail loudly with an `AnalysisException`/`KeyError` on customization rather than producing wrong numbers, so they are below the reporting bar.

### Issue A Resolution
- **Status**: Fixed
- **What changed**: The classification schemas now derive `Category = Literal[tuple(CATEGORIES)]` at runtime and use that type for both the judge's `expected_category` and the custom classifier's `category`.
- **Why**: This preserves a single customizable source of truth while restoring an exact structured-output enum, preventing case or spacing drift from corrupting metrics.
- **How verified**: Local Pydantic schema generation confirmed that the dynamic `Literal` emits the current categories as an enum. Notebook schema/AST validation and the complete Spark mock execution suite pass.

## Round 1 Final Verification

# Round 1 Broad-Sweep Verification — PySpark AI Functions Eval Notebooks

## Issues Found

**None.**

## Verdict

**PASS** — The staged diff is clean for correctness, security, logic, and user-spec conformance. No blocking or non-blocking issues identified.

---

## Evidence

### Scope
Staged diff is exactly six new files, +2,966 lines, no other tracked changes:
`AIFunctions-PySpark-eval-{analyze_sentiment,classify,extract,fix_grammar,summarize,translate}.ipynb` under `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/`.

### Runtime verification (strongest signal)
Executed **all six notebooks end-to-end, cell by cell, against a real local Spark 3.4.1 session** with a mocked AI accessor that enforces the documented public contract (rejects unknown kwargs, asserts `input_col` existence, validates every `{placeholder}` in `is_prompt_template=True` prompts against the actual DataFrame columns, and emits schema-conformant JSON derived from each supplied Pydantic model):

```
PASS AIFunctions-PySpark-eval-analyze_sentiment.ipynb
PASS AIFunctions-PySpark-eval-classify.ipynb
PASS AIFunctions-PySpark-eval-extract.ipynb
PASS AIFunctions-PySpark-eval-fix_grammar.ipynb
PASS AIFunctions-PySpark-eval-summarize.ipynb
PASS AIFunctions-PySpark-eval-translate.ipynb
```
Zero exceptions, zero unresolved template placeholders, zero unknown kwargs. This exercises the `.replace("{summary}"→"{concise_summary}")` / `"{translation}"→"{custom_translation}"` / `"{corrected}"→"{custom_corrected}"` prompt rewrites, the `sample_id` join in summarize §7, all sklearn/matplotlib paths, and every `toPandas()`.

### API conformance (verified against current Fabric PySpark docs)
- `deploymentName` / `reasoningEffort` are documented per-function camelCase overrides — matches `EXECUTOR_OPTIONS` / `JUDGE_OPTIONS`.
- Pydantic `BaseModel` is an explicitly documented `response_format` option for PySpark `ai.generate_response`.
- `labels=[aifunc.ExtractLabel(...), ...]` is a documented list form for PySpark `ai.extract` (with `label`/`description`/`type`/`max_items`).
- `ai.analyze_sentiment(labels=[...])`, `ai.summarize(instructions=...)`, `ai.summarize()` with `input_col` omitted (multi-column row synthesis), `ai.translate(to_lang=...)`, `ai.classify(labels=..., input_col=..., output_col=..., error_col=...)` all match documented signatures.
- All 7 documentation links return HTTP 200.

### Spec conformance audit
| Requirement | Status |
|---|---|
| Six PySpark counterparts | ✅ |
| `gpt-5-mini`/`low` executors, `gpt-5.1`/`medium` judges | ✅ AST audit: every `ai.*` call carries exactly one of `**EXECUTOR_OPTIONS` / `**JUDGE_OPTIONS`; all judge `generate_response` calls use `JUDGE_OPTIONS`, all function-under-test and custom-refinement calls use `EXECUTOR_OPTIONS` |
| No explicit API type | ✅ zero `apiType`/`api_type` occurrences |
| No temperature/top_p/seed | ✅ zero occurrences across all six |
| Shared verbosity unset; only optional summarize guidance | ✅ `verbosity` appears in **zero** code cells; only in summarize markdown ("leave the shared default unset") plus one non-executing illustrative snippet |
| Spark-native AI + judging | ✅ all AI transforms and judge calls are `df.ai.*` |
| Cached/materialized reused results | ✅ `materialize()` does `.cache()` + `.count()`; every downstream stage derives from an already-materialized frame |
| `ai.stats` | ✅ present in all six, on the materialized executor frame |
| Small final `toPandas()` | ✅ all conversions are `select(...)`-narrowed on ≤8-row result sets |
| Deterministic error columns | ✅ |
| Customizable categories from single source of truth | ✅ |
| 0–1 classification guidance | ✅ classify + analyze_sentiment use 0.80–1.00 / 0.70–0.79 / Below 0.70; the five-point-scale notebooks use 4.5–5.0 / 4.0–4.4 / 3.5–3.9 / Below 3.5 |
| Runnable custom sentiment labels & multi-column summarization | ✅ both execute (emotion/intensity/urgency schemes; `tickets_df.ai.summarize()` with no `input_col`) |
| No saved outputs / execution counts / empty cells | ✅ all cells `outputs: []`, `execution_count: null`, no empty cells; nbformat 4.5, no CRLF, no BOM |

### Deterministic error columns — isolated test
Ran `materialize()` on a frame with null / empty-string / whitespace-only / real-message / second-error-column rows:
```
2 row(s) contain AI Function errors:
detected rows: [4, 5]
no-id-col select ok: ['executor_error']
no-error-col path ok
```
`F.coalesce(F.length(F.trim(...)) > 0, F.lit(False))` yields no null-propagation and no whitespace false positives; multi-column OR-folding is correct.

### Classification single-source-of-truth — isolated test
Replacing `CATEGORY_GUIDANCE` with a different set and re-deriving `Category = Literal[tuple(CATEGORIES)]`:
```
eval enum   : ['refund', 'upgrade', 'churn_risk']
custom enum : ['refund', 'upgrade', 'churn_risk']
CATEGORIES  : ['refund', 'upgrade', 'churn_risk']
match       : True
```
Both the judge model (`ClassifyEval.expected_category`) and the custom classifier (`CustomClassifyResult.category`) stay constrained to the user-edited `CATEGORY_GUIDANCE`, and the same dict drives `_categories` / `_category_guidance` prompt columns. `ai.classify(labels=CATEGORIES)` shares the identical source. Single-element edge case (`Literal['only']`) also resolves.

### Parsing correctness — isolated test
`F.get_json_object(...).cast("boolean")` / `.cast("int")` on judge payloads:
```
  correct  score       exp
0    True    NaN  positive
1   False    NaN     mixed
2    None    5.0       NaN
3    None    NaN       NaN
```
Real booleans (not silent nulls), so `correct == True` / `.ne(True)` classify PASS/REVIEW correctly, and null responses fall to the review branch — the safe direction. Metric cells additionally guard with `dropna(subset=[...])`, an excluded-row count message, and a `raise ValueError` on an empty metric set.

### Security
No secrets, credentials, `subscription_key`/`URL` overrides, `%pip`/`%%sh`, `os.environ`, network fetches, `trust_remote_code`, or filesystem/table writes. All sample data is inline `spark.createDataFrame`; the only URLs are `learn.microsoft.com` doc links. The `password` / `requests.` grep hits are substrings of the sample customer-message text.

## Post-review Fabric Runtime Finding

- **Status**: Fixed
- **Finding**: Fabric runtime does not include Pydantic by default, so the six notebooks failed at `from pydantic import BaseModel, Field`.
- **What changed**: Every PySpark evaluation notebook now has `%pip install -q pydantic 2>/dev/null` as the first code cell, before the Pydantic import.
- **Why**: The pandas notebooks receive Pydantic transitively from their OpenAI install, while these PySpark notebooks need the dependency declared directly.
- **How verified**: Notebook validation asserts exactly one Pydantic install cell exists in each notebook and precedes the import. The full Spark mock execution suite passes with magic cells skipped by the local harness.

## Pydantic Fix Re-verification

## Issues Found

**None.**

## Verdict

**PASS** — The Pydantic install cell is correctly placed, uses the established Fabric notebook pattern, and introduces no regression.

## Evidence

- Each notebook has exactly one `%pip install -q pydantic 2>/dev/null` cell at index 2, which is the first executable cell.
- The Pydantic import follows at index 3 in all six notebooks, so the package is available before any `BaseModel` or `Field` use.
- Installing `pydantic` directly is the minimal dependency for the structured response schemas; the OpenAI SDK is not used by the PySpark notebook code.
- Runtime 1.3 already supplies the other imported packages used by these notebook paths.
- All notebook schemas validate, all non-magic code cells parse, all outputs and execution counts remain empty, and the full six-notebook Spark mock execution suite passes.
