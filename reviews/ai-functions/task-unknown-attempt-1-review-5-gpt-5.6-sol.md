## Review Summary

- **Round:** 5
- **Theme:** Testing and coverage
- **Mode:** sequential
- **Model:** gpt-5.6-sol
- **Issues Found:** 3
- **Verdict:** ISSUES_FOUND

## Issues

### Issue 1: Refinement deltas compare different row populations

- **Severity:** Medium
- **Files:** All six PySpark evaluation notebooks; comparison cells 18, 16, 15, 15, 17, and 15 respectively.
- **Problem:** Baseline and refined metrics independently discard null or failed rows, so a failed refinement row can falsely appear as a quality improvement.
- **Evidence:** With one failed custom row, sentiment reported baseline `0.5`, custom `1.0`, and delta `+0.5`; over the shared valid rows the delta was `0.0`. The same defect reproduced in grammar.
- **Suggested fix:** Compute both variants over one paired-valid mask, retain identifiers in comparison frames, and report paired and excluded row counts.

### Issue 2: Partially judged rows can be marked PASS

- **Severity:** Medium
- **Files:** Extraction, grammar, summarization, and translation result cells.
- **Problem:** Pandas row means skip missing judge scores. A row with one successful high score and missing remaining scores can be marked `PASS`.
- **Evidence:** Grammar scores `[5, null, null]` produced `average_score=5.0` and `status="PASS"`.
- **Suggested fix:** Require every expected metric before assigning `PASS`, `REVIEW`, or `FAIL`; otherwise mark the row `INCOMPLETE` and expose its scored-metric count.

### Issue 3: Structured judge scores are not range-constrained

- **Severity:** Medium
- **Files:** Extraction, grammar, summarization, and translation setup cells.
- **Problem:** `score: int = Field(description="Integer score from 1 to 5")` documents a range but does not enforce it. Values outside 1-5 can distort averages, statuses, and clipped charts.
- **Evidence:** The generated schema had no minimum or maximum and accepted both `0` and `6`.
- **Suggested fix:** Add enforceable 1-5 bounds and post-parse Spark validation that reports invalid values as failed or incomplete rows.

## Resolution Log

### Issue 1: Paired comparison populations

- **Changed:** Every baseline-versus-refinement comparison now retains `sample_id`, builds one shared paired-valid population, computes both variants over exactly those rows, reports paired and excluded counts, displays excluded identifiers when present, and fails explicitly if no paired rows remain.
- **Why:** A refinement failure can no longer improve its reported delta by disappearing from only one side of the comparison.
- **Verified:** A targeted sentiment case with one failed custom row now compares the one shared row and reports a zero accuracy delta rather than a false improvement. The complete notebook smoke suite also passes.

### Issue 2: Incomplete quality rows

- **Changed:** Quality result cells count non-null judge metrics per row, calculate an average only when every metric is present, and assign `INCOMPLETE` otherwise. Breakdowns expose `scored_metrics`, and aggregate metric statuses also become `INCOMPLETE` when any expected row is unscored.
- **Why:** Missing judge results are now visibly distinct from low or high quality scores.
- **Verified:** A targeted extraction result with scores `[5, null]` produces `scored_metrics=1`, a null average, and `status="INCOMPLETE"`.

### Issue 3: Enforced score bounds

- **Changed:** `MetricEval.score` now declares `ge=1` and `le=5`. `add_judge_metric()` additionally validates the parsed value in Spark, accepts only integral values from 1 through 5, nulls invalid scores, and appends an explicit error message to the metric error column.
- **Why:** Schema guidance and runtime metric inputs now enforce the same contract even if a provider response bypasses structured-output constraints.
- **Verified:** The generated schema includes minimum and maximum bounds; a mocked score of `6` becomes null and records `Judge score must be an integer from 1 to 5`.

### Final re-review

**CLEAN.** All comparison paths passed paired, excluded, outer-join, chart, and all-valid checks. Partial judgments produce `INCOMPLETE` with null averages, invalid scores are rejected with explicit errors, and all notebook schemas and Python cells validate.
