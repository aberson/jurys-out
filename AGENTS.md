# jurys-out

## Project overview

Public, local CLI utility for transparent judge-panel management and small,
reproducible code-review finding benchmarks. It tests whether a panel improves
human-adjudicated decisions over its best individual judge without hiding votes,
evidence, dissent, abstentions, or limits.

## Stack summary

| Layer | Tool |
|---|---|
| Runtime | Python >=3.12, uv, hatchling |
| CLI | argparse, no web backend or UI |
| Manifests | TOML |
| Data | JSONL events plus JSON and Markdown reports |
| Providers | OpenAI Responses and Anthropic Messages through narrow standard-library HTTP adapters |
| Checks | pytest, Ruff, strict mypy |

## Commands

~~~~powershell
uv sync --extra dev
uv run pytest -q
uv run ruff check .
uv run mypy --strict jurys_out
uv build
uv run jurys-out validate --panel panels/example-code-review.toml --suite suites/smoke.jsonl
uv run jurys-out preflight --panel panels/example-code-review.toml --suite suites/smoke.jsonl
uv run jurys-out run --panel panels/example-code-review.toml --suite suites/smoke.jsonl --max-requests 8 --max-input-bytes 20000 --max-output-tokens 600
uv run jurys-out report --run run_<uuidv7>
~~~~

## Directory layout

~~~~text
jurys_out/       CLI, contracts, adapters, runner, aggregation, analysis, reporting
judges/          Versioned Judge Cards
panels/          Versioned Panel Cards
suites/          Checked-in code-review finding benchmark cases
data/runs/       Local, Git-ignored run manifests, events, reports, raw outputs
docs/            Methodology and transparent card contract
tests/           Unit, integration, anchor, and invalid-input fixtures
~~~~

## Architecture summary

Judge and Panel Cards are strict TOML manifests. Benchmark cases are immutable
JSONL records with task/specification, diff, CI evidence, candidate finding, human
annotations, and final adjudication. The foreground runner freezes all input hashes,
writes append-only events, and resumes only exact-matching runs. It captures
structured votes and evidence for reports, keeps raw provider output local by
default, and marks a run incomplete rather than silently replacing an unavailable
judge. Aggregation is deterministic majority-with-escalation.

## Current state

Plan written, no code yet. Automated build Steps 1-7 and manual validation steps
M1-M2 are defined in plan.md.

## Environment requirements

Windows 11, PowerShell, uv, and Python >=3.12. OpenAI judges require
OPENAI_API_KEY in the active process environment; Anthropic judges require
ANTHROPIC_API_KEY. Never write secrets to manifests, run artifacts, reports,
tests, or Git. No port, server, background worker, or scheduler is required.
