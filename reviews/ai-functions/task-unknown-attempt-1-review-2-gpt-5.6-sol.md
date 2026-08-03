## Review Summary
- **Round**: 2
- **Theme**: Architecture & patterns
- **Mode**: sequential
- **Model**: gpt-5.6-sol
- **Artifact**: reviews/ai-functions/task-unknown-attempt-1-review-2-gpt-5.6-sol.md
- **Issues Found**: 4
- **Verdict**: ISSUES_FOUND

## Evidence Checklist
- [x] Inspected the actual Markdown and code source of every cell in all six staged notebooks.
- [x] Compared the notebooks with their pandas equivalents and the repository’s PySpark starter notebook.
- [x] Verified identical shared executor/judge options and absence of forbidden generation options.
- [x] Verified Spark-native AI processing, explicit error columns, narrowed `toPandas()` calls, and consistent shared helpers.
- [x] Verified no saved outputs, execution counts, empty cells, or staged notebook artifacts.
- [x] Compared chart inventories: every PySpark notebook has one chart, while its pandas equivalent has two or three.
- [x] Reproduced the conflicting-correctness-state problem: a row can report metric accuracy `1.0` while simultaneously appearing in the incorrect-results table.

## Issues

### Issue 1: Materialized AI results are not rebound before chained `.ai` calls
- **Severity**: High
- **File**:
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-analyze_sentiment.ipynb`
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb`
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-fix_grammar.ipynb`
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-summarize.ipynb`
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-translate.ipynb`
- **Cell**: Shared helper cell 3; downstream chaining in cells 9/12, 10, 9/14, 9/16, and 9/14 respectively.
- **Problem**: `materialize()` returns the same cached DataFrame object. The repository’s Runtime 1.3 PySpark convention explicitly uses `.cache().select("*")` before chaining another AI Function so the `.ai` accessor binds to the generated schema.
- **Evidence**: `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/starter-notebooks/AIFunctions-PySpark-starter-notebook.ipynb`, cells 4–6 and 21, documents and demonstrates this rebind pattern.
- **Risk**: Chained judging or refinement can bind to the pre-transformation schema, fail to resolve generated columns such as `sentiment`, `category`, `corrected`, `summary`, or `translation`, or drop preceding generated columns.
- **Fix**: Preserve the original materialized frame for `ai.stats`, but create a fresh `.select("*")` view before every downstream `.ai` call. Encapsulate this distinction in the shared helper or use separate result and chaining variables.

### Issue 2: The classification judge does not consume the canonical category guidance
- **Severity**: Medium
- **File**: `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb`
- **Cell**: 5, 9, 15
- **Problem**: `CATEGORY_GUIDANCE` supplies the custom classifier’s definitions, but the judge receives only `_categories`. Consequently, expected labels and custom predictions are generated under different category contracts.
- **Risk**: Ambiguous cases can be judged using category-name intuition rather than the definitions used by the custom classifier, invalidating baseline/custom metric comparisons.
- **Fix**: Include `_category_guidance` in `EVAL_PROMPT` so the judge and custom classifier share the same canonical definitions while retaining `CATEGORIES` as the `Literal` source.

### Issue 3: Correctness has two independent sources of truth
- **Severity**: Medium
- **File**:
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-analyze_sentiment.ipynb`
  - `/Users/rana-ms-work/Documents/fabric-samples/docs-samples/data-science/ai-functions/eval-notebooks/AIFunctions-PySpark-eval-classify.ipynb`
- **Cell**: 11–15 and 9–13 respectively
- **Problem**: The judge returns both `correct` and an expected label. Metrics compare predicted and expected labels, while breakdown tables trust `correct`; no invariant enforces agreement.
- **Risk**: The notebook can display a row as incorrect while counting it as correct in accuracy, or vice versa.
- **Fix**: Remove `correct` from the response schema and derive it in Spark from the predicted and expected labels. Use that derived column everywhere.

### Issue 4: The PySpark equivalents omit required pandas chart coverage
- **Severity**: Medium
- **File**: All six staged PySpark evaluation notebooks
- **Cell**: Result/refinement cells 14/18, 12/16, 11/15, 11/15, 11/17, and 11/15 respectively
- **Problem**: Each PySpark notebook retains only one aggregate bar chart. The pandas equivalents additionally include distributions, radar/histogram diagnostics, accuracy pies, and baseline-versus-refinement charts.
- **Risk**: The notebooks are not equivalent to the pandas samples and omit the visual diagnostics required for evaluating optional refinement paths.
- **Fix**: Restore the corresponding chart intents using the already narrowed pandas result frames; do not broaden Spark-to-pandas conversion.

## Resolution Log

### Issue 1: Fresh DataFrame boundaries

- **Changed:** Added `fresh_ai_view(frame)` and use it before every AI call that consumes an earlier AI transformation, including iterative judge loops and optional comparison paths.
- **Why:** This follows the starter notebook's supported chaining pattern while retaining the complete generated row shape.
- **Verified:** Regenerated all six notebooks and executed every non-magic cell with local Spark and mocked AI Functions.

### Issue 2: Canonical classification guidance

- **Changed:** Added one `CATEGORY_GUIDANCE` mapping, derive `CATEGORIES` from it, attach formatted definitions as `_category_guidance`, and include that column in both baseline and custom judge prompts.
- **Why:** The classifier and judge now share the same user-editable category contract without duplicated definitions.
- **Verified:** Confirmed both judge paths receive the canonical guidance and the full classification notebook executes successfully.

### Issue 3: Deterministic correctness

- **Changed:** Removed judge-returned `correct` fields from sentiment and classification response schemas. Both notebooks derive `correct` in Spark by comparing predicted and expected labels; the judge returns only explanatory text.
- **Why:** Exact label equality is deterministic and should not be independently decided by an LLM.
- **Verified:** Confirmed the derived Boolean drives accuracy, error analysis, and comparison metrics in both notebooks.

### Issue 4: Visual diagnostic parity

- **Changed:** Added accuracy pies for sentiment and classification; score distributions and radar charts for extraction, grammar, summarization, and translation; grouped comparison charts for every optional refinement path; and side-by-side summarization conciseness/length charts.
- **Why:** These retain the useful pandas evaluation diagnostics while keeping plotting inputs narrow and notebook-specific.
- **Verified:** Executed all chart cells with a non-interactive Matplotlib backend during the complete notebook smoke run.

### Re-review finding: Iterative calls also require fresh views

- **Finding:** The first re-review found that iterative judge loops and the custom sentiment-label loop refreshed only before entering the loop, not before each successive AI call.
- **Changed:** `add_judge_metric()` now calls `fresh_ai_view(frame)` internally on every invocation, and each custom sentiment-label iteration refreshes its current frame before calling `analyze_sentiment`.
- **Why:** Every downstream AI transformation now receives a new DataFrame view, including the second and later transformations in a loop.
- **Verified:** Regenerated the notebooks and reran the complete mocked Spark execution suite successfully.

### Final re-review

**CLEAN.** The reviewer confirmed fresh views on every downstream call and loop iteration, shared classification guidance, deterministic correctness, and pandas-equivalent diagnostic coverage with no remaining architecture findings.
