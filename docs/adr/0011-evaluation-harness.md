# ADR-0011: An evaluation harness with no evaluators

- **Status:** Accepted
- **Date:** 2026-08-26

## Context
In ordinary software a test passes or fails. In a generative system "correct"
is a distribution: the same input can produce a different answer each time, and
a change that improves one behaviour quietly degrades another. The unit test is
still necessary and is no longer sufficient.

So the template needs an evaluation harness. It does **not** need evaluators --
groundedness belongs to a retrieval service, task success and trajectory
scoring to an agent runtime, and a template that guessed would be wrong for
both. Same reasoning as ADR-0010.

## Decision
Ship the harness, the dataset loader, the threshold gate and the reporters.
Ship no metrics. Five decisions inside that are worth stating.

### 1. A golden set is identified by the hash of its contents
Not a version string someone remembers to bump. Every run records the SHA-256
of the dataset file, and the reports print it.

This is the subtlety that defeats most eval setups. Scores are only comparable
across runs of the *same* dataset. Add ten cases and today's 0.91 is not better
than last week's 0.88 -- it is a different measurement wearing the same name.
Hashing makes a quiet dataset edit visible instead of letting it look like a
quality improvement. The gate **refuses** a baseline comparison when the hashes
differ, and says so in its notes, rather than producing a confident wrong answer.

### 2. Every score carries a value *and* a pass/fail
A mean of 0.82 says the average is fine and hides that one case in six is
catastrophic. `Threshold` therefore accepts `min_mean` and `min_pass_rate`
together, and there is a test asserting a healthy mean cannot hide a bad pass
rate. For a regulated use case, the failing sixth is the only part that matters.

### 3. Absolute floors and regression checks are both required
An absolute floor catches a system that was never good enough. It never notices
a system sliding from 0.95 to 0.86, because 0.86 still clears the bar.
`max_regression` catches the decay. Neither alone is enough.

### 4. Nothing raised inside a run can abort it
A target that throws is recorded as an error result; an evaluator that throws
produces a failing score with the exception in its detail. Losing 199 cases
because one raised is the worst possible trade -- and an evaluator bug must not
be able to make a case silently disappear from the report.

### 5. Cost and latency are first-class, not an appendix
Every case records latency, and the gate takes a p95 ceiling. A system that is
three points more accurate and four times more expensive is usually worse, and
a report showing only quality cannot tell you that. When services add model
spans (ADR-0008), cost per case joins latency through the same path.

## Alternatives considered
- **Shipping default metrics.** Wrong for most services, and wrong defaults are
  worse than none because people assume they are covered.
- **A version string on the dataset.** Relies on discipline exactly when someone
  is in a hurry, which is when datasets get edited.
- **Reporting means only.** The industry default, and the reason systems ship
  with known failure modes nobody noticed.
- **Unbounded concurrency.** The target is usually a rate-limited API; firing
  200 cases at once produces a report full of 429s that says nothing about
  quality. Hence `concurrency=8` by default.
- **Interpolated percentiles.** On a 50-case golden set, interpolation invents a
  latency no case actually produced. Nearest-rank is the honest one at this size.
- **`python -m evals` printing "not configured" and exiting 0.** A service that
  wires the eval job into CI before writing an evaluator would get a permanently
  green, permanently meaningless check. It exits 1.

## Consequences
A service defines its own `Evaluator`s and its own thresholds -- its Production
Quality Bar -- and calls `run_eval` from its own entry point. What the template
guarantees is that every service reports the same shape, records which dataset
produced a number, and can fail a build on quality rather than merely reporting
on it. Growing the golden set changes its hash and resets the baseline; that is
correct, and the notes in the gate output say so out loud.
