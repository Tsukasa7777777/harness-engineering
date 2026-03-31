# 04_evals — Evaluation Framework

This section defines how to measure, test, and improve agent quality. Evaluations are the engineering discipline that turns agent development from guesswork into a rigorous, data-driven practice.

## Contents

| File | Purpose |
|------|---------|
| `eval-framework.md` | Core evaluation methodology and metric definitions |
| `benchmark-suite.md` | Standardized benchmarks for common agent capabilities |
| `regression-tests.md` | Tests that catch quality degradation after changes |

## Philosophy

> "If you can't measure it, you can't improve it."

Every harness change should be validated against evaluations before deployment. The eval suite serves as both a quality gate and a feedback mechanism for continuous improvement.

## Quick Start

1. Read `eval-framework.md` to understand the evaluation methodology
2. Select relevant benchmarks from `benchmark-suite.md`
3. Set up regression tests from `regression-tests.md`
4. Run evals after every harness configuration change
