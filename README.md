# Jurys Out

> Codifying judge panels for agentic review workflows, because the jury is still out.

Jurys Out is a public, local command-line utility for defining transparent judge
panels and running small, reproducible benchmarks for agentic code-review
findings. It is designed to test whether a panel improves human-adjudicated
decisions over its best individual judge without hiding votes, evidence, dissent,
abstentions, costs, or limits.

## Status

This repository currently contains the canonical implementation plan. The CLI and
its example fixtures will be built in the tracked implementation steps; see
[plan.md](plan.md) for the full contract and delivery order.

## What it measures

Jurys Out is a measurement instrument, not a leaderboard. It evaluates
code-review finding validation against human adjudications, using calibration
anchors, frozen holdouts, and invariance fixtures. A panel is considered better
only when it improves the selected held-out metric over its best constituent judge
under comparable limits.

## Stack

| Layer | Tool | Purpose |
| --- | --- | --- |
| Runtime | Python 3.12+ | Modern typing and standard-library TOML support. |
| Packaging | uv + hatchling | Lightweight, reproducible CLI environment. |
| CLI | argparse | Scriptable command interface with no web surface. |
| Manifests and data | TOML, JSONL, JSON, Markdown | Reviewable cards, append-friendly events, and transparent reports. |
| Provider transport | urllib.request + json | Narrow, dependency-free OpenAI and Anthropic adapters. |
| Quality gates | pytest, Ruff, strict mypy | Tests, linting, and static type checks. |

## Scope

Version 1 is a foreground, local CLI. It deliberately excludes a web UI, server,
database, queue, worker, scheduler, multi-user service, CI merge gate, automatic
panel optimizer, external-data scraper, and broad provider plug-in marketplace.

## Planned layout

```text
jurys-out/
  jurys_out/       CLI, contracts, adapters, runner, aggregation, analysis, reports
  judges/          Versioned Judge Cards
  panels/          Versioned Panel Cards
  pricing/         Versioned local price book
  suites/          Immutable benchmark-case suites
  data/runs/       Local, Git-ignored manifests, events, reports, and raw outputs
  docs/            Methodology and transparent card contracts
  tests/           Unit, integration, anchor, and invalid-input fixtures
```

## Prerequisites

- Python 3.12 or later
- [uv](https://docs.astral.sh/uv/)
- An OpenAI or Anthropic API key only for an optional live provider smoke test

Never place API keys in cards, suites, reports, tests, Git, or any other project
file. Live runs read `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` only from the active
shell environment.

## Development setup

The following commands become available after the package foundation is built.

1. Clone the repository and enter it.

   ```powershell
   git clone https://github.com/aberson/jurys-out.git
   cd jurys-out
   ```

2. Create the development environment.

   ```powershell
   uv sync --extra dev
   ```

3. Run the automated quality gates.

   ```powershell
   uv run pytest -q
   uv run ruff check .
   uv run mypy --strict jurys_out
   uv build
   ```

4. Validate the checked-in example panel and suite.

   ```powershell
   uv run jurys-out validate --panel panels/example-code-review.toml --suite suites/smoke.jsonl
   uv run jurys-out preflight --panel panels/example-code-review.toml --suite suites/smoke.jsonl
   ```

## Intended workflow

1. Define versioned Judge Cards with an explicit role, rubric, overlap, abstention
   policy, and structured `finding-v1` output.
2. Assemble those cards into a Panel Card using deterministic
   `majority_with_escalation` aggregation.
3. Validate a local JSONL benchmark suite with immutable, human-adjudicated cases.
4. Run the panel within explicit request, input-byte, output-token, and optional
   estimated-cost limits.
5. Generate deterministic JSON and Markdown reports from the frozen manifest and
   append-only events.

## Transparency and safety

- Every run freezes panel, judge, suite, and execution-limit hashes before a model
  call.
- Events are append-only; interrupted runs remain inspectable and may resume only
  when every frozen input and limit exactly matches.
- Required-judge failures make a run incomplete rather than producing substituted
  votes or a fabricated panel score.
- Ties, all-abstain results, and missing required evidence escalate. Reports retain
  individual votes, evidence references, abstentions, and dissent.
- Raw provider responses stay local by default under `data/runs/`, are Git-ignored,
  and are excluded from public reports unless explicitly exported.
- Automated tests use a deterministic fake adapter and make no live model calls.

## Key design decisions

- **Complementarity is empirical.** Roles state their intended ownership and
  overlap; the benchmark measures shared errors rather than assuming a perfectly
  partitioned panel.
- **Measurement validity precedes selection.** Calibration anchors, frozen
  holdouts, ambiguous human outcomes, and invariance checks constrain claims.
- **Provider parity without false uniformity.** OpenAI Responses and Anthropic
  Messages use one internal protocol while preserving provider-specific behavior in
  local raw artifacts and reports.
- **Limits are explicit.** Request, input-byte, and output-token ceilings always
  apply. Estimated USD limits require a versioned local price book.

## License

The project is planned for the MIT License; the license file is part of the first
implementation step.
