# jurys-out plan

> **Toolkit program authority — 2026-09-24.** For units mapped from this document, the [toolkit program](../plan.md) owns selection, shared sequencing and current execution/status; the [source-unit ledger](../documentation/toolkit-program-units.md) identifies that mapped scope. Status and execution instructions retained below for transferred units are source history, not a second dispatch queue. Technical specifications, original IDs, acceptance criteria and evidence remain owned here. Update transferred-unit status in the program only; unmapped local work remains locally owned. Mapping does not complete, reopen or authorize a unit.

## 1. What This Is

Jurys Out is a public, MIT-licensed, local CLI utility for defining transparent judge panels and
running small, reproducible benchmarks that show whether a panel improves
human-adjudicated agentic code-review finding decisions over its best individual
judge. Its tagline is: "Codifying judge panels for agentic review workflows,
because the jury is still out."

Proposal: documentation/jurys-out-proposal.html

The first release is deliberately a thin, foreground tool. It validates versioned
Judge Cards and Panel Cards, runs a selected panel against a local benchmark suite,
records each judge's structured verdict and evidence, and reports quality,
disagreement, cost/latency, and correlated-error diagnostics. It has no web UI,
multi-user service, scheduler, distributed worker, CI merge gate, automatic panel
optimizer, external-dataset scraper, or broad provider plug-in marketplace.

## 2. Stack

| Layer | Tool | Why |
|---|---|---|
| Runtime | Python >=3.12 | Matches the workspace benchmarking baseline and supplies tomllib and modern typing. |
| Environment and packaging | uv + hatchling | Verified local convention in ../measure-twice/pyproject.toml; supports a small installable CLI. |
| CLI | argparse in the standard library | Keeps the first release dependency-light and scriptable. |
| Manifests | TOML | Human-reviewable, Git-friendly, and readable through tomllib without a YAML dependency. |
| Benchmark and run data | JSONL plus JSON and Markdown summaries | Append-friendly, auditable, and consistent with the local benchmark pattern in ../measure-twice/CLAUDE.md. |
| Provider transport | urllib.request and json in the standard library | Keeps two narrow direct HTTP adapters dependency-free. |
| OpenAI judge adapter | Responses API, POST /responses, structured JSON output, store=false | The official API accepts text or JSON outputs and exposes status, errors, incomplete details, and usage. See [OpenAI Responses API](https://developers.openai.com/api/reference/cli/resources/responses/methods/create). |
| Anthropic judge adapter | Messages API, POST /v1/messages, output_config.format JSON Schema | The official API accepts structured messages and a JSON Schema output format. See [Anthropic Messages API](https://platform.claude.com/docs/en/api/messages/create) and [Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs). |
| Tests and static checks | pytest, Ruff, mypy --strict | Matches the verified measure-twice toolchain: pytest, Ruff checks, and strict mypy. |

No server, port, browser surface, database, queue, or background task is part of
version 1.

Terms used below: CLI means command-line interface; CI means continuous
integration; JSONL means newline-delimited JSON; API means application programming
interface; HTTP means Hypertext Transfer Protocol; UUID means universally unique
identifier; RFC means an Internet standards document; and F1 is the harmonic mean
of precision and recall.

## 3. Data Store

All persistent data is local and file based.

| Path | Contents | Integrity rule |
|---|---|---|
| judges/<judge_id>.toml | Versioned Judge Cards | Strict schema; unknown keys and invalid IDs are rejected. |
| panels/<panel_id>.toml | Versioned Panel Cards | Every referenced judge and its manifest hash must resolve before a run starts. |
| suites/<suite_id>.jsonl | Immutable BenchmarkCase records | Each line is validated independently; duplicate case IDs are rejected. |
| pricing/price-book.toml | Versioned PriceBook entries for estimated USD limits | Strict schema; every provider/model pair has non-negative input and output rates. |
| data/runs/<run_id>/manifest.json | Frozen run manifest: panel, judge hashes, suite hash, options, timestamps | Written atomically before any model call. |
| data/runs/<run_id>/events.jsonl | Append-only case-by-judge events, including failures | Each completed event is durable before the next call; resume reuses only exact matching hashes. |
| data/runs/<run_id>/summary.json | Deterministic metrics and aggregation | Regenerated from events; never the only source of truth. |
| data/runs/<run_id>/report.md | Transparent human-readable report | Regenerated from the frozen run manifest and events. |
| data/runs/<run_id>/raw/<case_id>/<judge_id>.json | Full raw provider response, retained locally by default | Git-ignored and never included in a public report unless explicitly exported. |

IDs are concrete and stable:

- Judge and panel IDs match ^[a-z][a-z0-9-]{1,62}$.
- A case ID is case-sha256- followed by the 64 lowercase hexadecimal characters
  of SHA-256 over its canonical JSON object excluding only its own case_id field.
  Altering the task, diff, evidence, finding, or adjudication creates a new case.
- A finding ID is finding-sha256- followed by SHA-256 of the canonical
  case_id, claim, and declared severity.
- A run ID is run_ followed by an RFC 9562 UUIDv4 string.
- Every manifest, suite, and raw artifact also records a sha256 hex digest.

Writes use a sibling temporary file followed by atomic replacement for JSON, TOML,
and report files. JSONL events are append-only; an interrupted run remains
inspectable. Resume is allowed only when the suite hash, panel hash, all Judge Card
hashes, and execution limits exactly match. Otherwise the CLI refuses resume and
requires a new run ID.

`RunManifest` shape:

| field | type | note |
|---|---|---|
| run_id | string | The generated `run_` identifier for this attempt. |
| panel_hash, suite_hash | lowercase SHA-256 strings | Freeze the exact panel and suite. |
| judge_hashes | object | Maps each judge ID to its card digest. |
| limits | object | Carries request, input-byte, output-token, and any active cost ceilings. |
| started_at | RFC 3339 timestamp | Records when execution began. |

`RunEvent` shape:

| field | type | note |
|---|---|---|
| run_id, case_id, judge_id | strings | Identify one durable case-by-judge attempt. |
| attempt | positive integer | Records the initial call or a permitted retry. |
| status | `completed`, `abstained`, or typed failure | Never disguises a failure as a vote. |
| result or failure | finding-v1 result or typed error | Stores a validated result or redacted diagnostic facts. |
| started_at, completed_at | RFC 3339 timestamps | Support transparent latency reporting. |

`PriceBook` shape:

| field | type | note |
|---|---|---|
| schema_version | integer | Starts at 1. |
| currency | literal `USD` | The only currency supported in version 1. |
| entries | array | One entry per exact provider/model pair. |
| entries[].provider, entries[].model | strings | Must match a configured Judge Card exactly. |
| entries[].input_usd_per_1m, entries[].output_usd_per_1m | non-negative numbers | Token prices used only for a preflight estimate. |

## 4. Judge Cards and Panel Cards

### Judge Card

A Judge Card is transparent configuration, not hidden prompt logic. It contains:

- schema_version = 1 and the judge ID;
- provider, exact model string, and provider API version/header requirements;
- role, primary review dimension, explicit exclusions, and declared overlap
  dimensions;
- a fixed rubric and evidence requirement embedded in the manifest;
- structured output schema version finding-v1;
- abstention conditions, token/output limits, and retry_policy = transient-v1; and
- a card digest recorded in every run.

Judges are not required to be mutually exclusive or collectively exhaustive.
Instead, a card must state its primary ownership and intentional overlap. The
tool reports empirical redundant errors rather than inferring redundancy from role
names alone.

### Panel Card

A Panel Card names ordered Judge Card references and a deterministic
majority_with_escalation aggregation rule.

- All configured judges must complete for a benchmark run to be marked complete.
  A transport error, provider refusal, schema violation, or exhausted retry budget
  leaves the run incomplete; the partial events remain available for diagnosis.
- A declared abstention is a valid judge outcome, not a successful valid/invalid
  vote.
- For a completed case, the panel verdict is the strict majority of non-abstaining
  valid and invalid votes. A tie, no non-abstaining vote, or a missing required
  evidence reference yields escalate.
- The report always shows the individual votes, evidence references, abstentions,
  and dissent. It does not expose private model reasoning traces.

## 5. Benchmark Cases and Runner Lifecycle

Version 1 evaluates code-review finding validation only. Each BenchmarkCase has:

- a required split of calibration_good, calibration_garbage, holdout, or sentinel;
- a task or specification;
- a patch or diff;
- relevant test or CI evidence;
- one candidate finding with a claim and severity;
- independent human annotations; and
- a final human adjudication of valid, invalid, or ambiguous, with an adjudication
  rationale.

`BenchmarkCase` shape:

| field | type | note |
|---|---|---|
| case_id | deterministic case-sha256 string | Derived from the other case content. |
| split | required enum | calibration_good, calibration_garbage, holdout, or sentinel. |
| task_spec, diff | strings | The review task and the change under review. |
| evidence | array of artifact records | Relevant tests or CI output with stable locations. |
| candidate_finding | object | Claim, declared severity, and deterministic finding ID. |
| human_annotations | array | Independent annotations retained for audit. |
| adjudication | object | valid, invalid, or ambiguous, with rationale. |

The runner sends the same case material to every configured judge as delimited,
untrusted data. The fixed judge instruction says that embedded source text, tests,
CI logs, and candidate findings are evidence to analyze, never instructions to
follow. No fetched URLs, issue bodies, or external tools are included in version 1.

The runner is foreground only. It performs preflight validation, writes the frozen
run manifest, executes one case-by-judge call at a time, records every event, and
prints progress. Under retry_policy = transient-v1, it retries a timeout,
connection failure, HTTP 429, or HTTP 5xx response at most twice, respects
Retry-After when supplied, and records each attempt. It never retries authentication,
local validation, provider refusal, or another HTTP 4xx response, and never silently
substitutes a model or provider.

Execution limits are explicit: maximum requests, maximum input bytes per case,
maximum output tokens per request, and an optional maximum estimated USD cost. When
--max-estimated-cost-usd is supplied, pricing/price-book.toml is required at
preflight and must contain every exact provider/model pair; otherwise the run is
refused before a model call. Without that option, the report labels a dollar
estimate unavailable rather than presenting a false value. Request and token
ceilings are always enforced, including during manual live checks.

## 6. Analysis, Calibration, and Transparency

The benchmark is a measurement instrument, not a leaderboard. Every case declares
one split: calibration_good, calibration_garbage, holdout, or sentinel. Calibration
contains known-good and known-garbage anchors; the tool refuses to report a judge
metric as calibrated unless score(good) is greater than score(garbage). Only the
frozen holdout supports a panel-selection claim; sentinels recur unchanged to expose
regressions.

For each judge and completed panel, the report computes:

- macro F1 against non-ambiguous human adjudications;
- false-valid and false-invalid rates;
- abstention and escalation rates;
- latency and provider-reported token usage per completed call;
- pairwise agreement, joint-error count, double-fault rate, and
  difficulty-conditioned residual error correlation where sample size permits;
- unique verified findings and leave-one-out marginal panel contribution; and
- comparison with the best individual judge under the same frozen suite and limits.

The fixture suite includes semantic-preserving order-swap and identifier-renaming
variants, plus a misleading-comment variant. Their expected adjudication is the
same as the source case; an unexplained verdict change is reported as an invariance
failure rather than a quality improvement.

Insufficient data is reported as insufficient data, never as zero correlation or a
quality claim. Ambiguous cases remain visible but do not enter binary F1. The report
does not declare a panel better merely because it contains more judges; it must
improve the selected held-out metric over the best individual judge at comparable
limits.

## 7. Modules

### jurys_out/

| File | Responsibility |
|---|---|
| __init__.py | Package version and public constants. |
| cli.py | argparse command dispatch for validate, preflight, run, report, and smoke. |
| errors.py | Typed, actionable domain errors and stable exit codes. |
| ids.py | Canonical JSON serialization, SHA-256 IDs, UUIDv4 generation, and digest helpers. |
| contracts.py | Strict in-memory representations for cards, cases, events, and reports. |
| manifests.py | TOML loading, strict key validation, cross-reference checks, and manifest hashing. |
| suites.py | JSONL loading, case validation, suite hashing, and calibration/holdout partition checks. |
| storage.py | Atomic file writes, append-only events, raw-artifact paths, and exact-match resume. |
| runner.py | Foreground execution loop, limits, retry classification, and incomplete-run handling. |
| aggregation.py | majority_with_escalation aggregation and dissent preservation. |
| analysis.py | Metrics, anchors, error-correlation diagnostics, and best-single comparison. |
| report.py | Deterministic JSON and Markdown transparency reports. |

### jurys_out/adapters/

| File | Responsibility |
|---|---|
| base.py | Provider-neutral request/result protocol and retry classification. |
| fake.py | Deterministic test adapter; no network or credential access. |
| openai.py | OpenAI Responses request/response mapping, with structured output and store=false. |
| anthropic.py | Anthropic Messages request/response mapping with output_config.format JSON Schema. |

## 8. Project Structure

~~~~text
jurys-out/
  AGENTS.md                 # Project instructions generated by plan-init
  CLAUDE.md                 # Exact @AGENTS.md pointer
  plan.md                   # This canonical, observer-scrapable plan
  LICENSE                   # MIT license for the public release
  README.md                 # Public introduction, quickstart, and transparency policy
  pyproject.toml            # Python package, CLI entry point, lint/type/test configuration
  jurys_out/                # CLI, contracts, runner, analysis, reporting
    adapters/               # Fake, OpenAI, and Anthropic adapters
  judges/                   # Versioned example Judge Cards
  panels/                   # Versioned example Panel Cards
  pricing/
    price-book.toml         # Versioned local USD estimate inputs
  suites/                   # Checked-in smoke and calibrated fixture suites
  data/
    runs/                   # Git-ignored local run records and raw responses
  docs/
    methodology.md          # Measurement claim, suite policy, limitations
    judge-card-schema.md    # Transparent field and output contract
  tests/
    anchors/                # Frozen known-good/known-garbage calibration fixtures
    fixtures/               # Invalid manifests, cases, fake transport transcripts
~~~~

## 9. Key Design Decisions

### Complementarity over rigid MECE

Judge roles use clear primary ownership and declared overlap rather than a claim of
perfect partitioning. Deliberate overlap is useful for recall and calibration; what
the tool measures is shared error on human-adjudicated cases. This makes the
five-judge question empirical rather than doctrinal.

### Local, reproducible benchmark execution

The runner is in scope because panel management alone cannot establish whether a
panel helps. It stays thin by accepting local fixtures and manifests, executing in
the foreground, and avoiding worker infrastructure. The frozen manifest plus
append-only events make a result replayable and auditable.

### Transparent structured outputs and local raw audit data

Every public report is built from structured votes, quoted evidence locations,
concise rationales, configuration digests, and dissent. Full raw provider results
are retained locally by default to diagnose parser and correlation failures, but
are Git-ignored and require an explicit export to leave the machine.

### Provider parity without false uniformity

OpenAI and Anthropic are the first direct adapters because cross-family comparison
is meaningful for the product. Their HTTP contracts are isolated behind the same
internal result protocol, but provider-specific model names, usage fields, response
status, and schema behavior stay visible in the raw artifact and report.

### Deterministic aggregation and explicit escalation

The first aggregation rule is intentionally simple. It preserves individual
decisions, has no opaque debate phase, and escalates ties and abstentions instead
of inventing confidence. More elaborate aggregation is future work and must prove
its advantage on a frozen holdout.

### Measurement validity before selection

Human adjudication, calibration anchors, frozen holdouts, end-to-end code-review
artifacts, and invariance fixtures are part of the product contract. A panel is
selected only when evidence supports a concrete claim; no score is allowed to gate
a workflow merely because it is easy to compute.

## 10. Open Questions and Risks

| Item | Risk | Mitigation |
|---|---|---|
| Small initial suite | Correlation and quality metrics can be noisy. | Label insufficient samples, retain sentinels, and require holdout improvement before a selection claim. |
| Sensitive diffs and raw output | Local benchmark contents or responses may be confidential. | Git-ignore data/runs, redact reports by default, support --no-retain-raw, and never write credentials. |
| Provider schema/refusal behavior | A structured-output request can still be refused or incomplete. | Validate every response locally; record the specific outcome; mark a required-judge failure incomplete. |
| Price changes | Dollar estimates can become stale. | Require a versioned local price book for an estimated-USD ceiling; always enforce request/token ceilings and label estimates unavailable when no ceiling is requested. |
| Prompt injection in review artifacts | A diff or CI log can contain instructions targeting the judge. | Delimit artifacts as untrusted data, use a fixed rubric, do not enable tools, and include adversarial fixtures. |
| Shared model-family error | Different roles can still make the same error. | Show error correlation and double faults; compare the panel to the best individual judge. |
| Human label disagreement | A forced binary gold label can misrepresent contested reviews. | Preserve all annotations, represent ambiguous adjudication, and exclude ambiguity from binary F1. |

## 11. How to Run

After the implementation steps are complete:

~~~~powershell
cd C:\Users\abero\dev\jurys-out
uv sync --extra dev
uv run jurys-out --help
uv run pytest -q
uv run ruff check .
uv run mypy --strict jurys_out
uv build
uv run jurys-out validate --panel panels/example-code-review.toml --suite suites/smoke.jsonl
uv run jurys-out preflight --panel panels/example-code-review.toml --suite suites/smoke.jsonl
uv run jurys-out run --panel panels/example-code-review.toml --suite suites/smoke.jsonl --max-requests 8 --max-input-bytes 20000 --max-output-tokens 600
uv run jurys-out report --run run_<uuidv4>
~~~~

For a real provider run, set only the required environment variables in the active
shell. The tool reads OPENAI_API_KEY for OpenAI judges and ANTHROPIC_API_KEY for
Anthropic judges. Do not add either value to TOML, JSONL, reports, tests, or Git.

## 12. Development Process

Build with /build-phase using worktree isolation. Code and data-integrity steps use
the default code review lane; network, credential, and persistence seams use the
deeper code review lane. No step needs runtime UI reviewers because the product is
a CLI. Automated tests use the fake adapter and never send a live model request.

## Build Steps

### Automated Steps

### Step 1: Package and CLI foundation
- **Status:** TODO
- **Problem:** Create the installable Python package, MIT license, standard development checks, command skeleton, and Git-ignore policy for local run artifacts.
- **Type:** code
- **Issue:** #
- **Files:** pyproject.toml, .gitignore, LICENSE, README.md, jurys_out/__init__.py, jurys_out/cli.py, tests/test_cli.py
- **Flags:** --reviewers code --isolation worktree
- **Produces:** A buildable, MIT-licensed jurys-out CLI whose help, validate, preflight, run, report, and smoke commands have stable exit-code placeholders.
- **Done when:** uv build, pytest, Ruff, and strict mypy pass; data/runs and raw response artifacts are ignored by Git.
- **Depends on:** none

### Step 2: Strict cards, cases, and suite validation
- **Status:** TODO
- **Problem:** Implement the versioned TOML and JSONL contracts, canonical IDs, hashing, unknown-key rejection, PriceBook validation, explicit case splits, and fixture validation for Judge Cards, Panel Cards, BenchmarkCases, and adjudications.
- **Type:** code
- **Issue:** #
- **Files:** jurys_out/contracts.py, jurys_out/ids.py, jurys_out/manifests.py, jurys_out/suites.py, judges/example-evidence.toml, panels/example-code-review.toml, pricing/price-book.toml, suites/smoke.jsonl, tests/test_contracts.py, tests/test_manifests.py, tests/test_suites.py
- **Flags:** --reviewers code --isolation worktree
- **Produces:** A validate command that accepts the documented fixtures and rejects duplicate IDs, invalid hashes, unknown fields, malformed price entries, bad cross-references, invalid case splits, and malformed human adjudications.
- **Done when:** Fixture tests prove every accepted record has a concrete ID and split, and every invalid fixture fails with a stable error code.
- **Depends on:** 1

### Step 3: Fake end-to-end runner with durable resume
- **Status:** TODO
- **Problem:** Build the provider-neutral adapter protocol, deterministic fake adapter, atomic run manifest, append-only events, execution and estimated-USD ceilings, transient-v1 retry classification, and exact-match resume behavior.
- **Type:** code
- **Issue:** #
- **Files:** jurys_out/adapters/base.py, jurys_out/adapters/fake.py, jurys_out/storage.py, jurys_out/runner.py, tests/test_runner.py, tests/test_storage.py, tests/fixtures/
- **Flags:** --reviewers deep --isolation worktree
- **Produces:** A fake-adapter run that survives an injected interruption, enforces an estimated-USD ceiling only with a complete PriceBook, records every retry attempt, and resumes only when every frozen input hash matches.
- **Done when:** An integration test runs a real CLI process against the fake adapter, interrupts it, resumes it, proves no case-by-judge event is duplicated, and verifies the transient-v1 retry boundary.
- **Depends on:** 2

### Step 4: Deterministic aggregation and measurement report
- **Status:** TODO
- **Problem:** Implement majority_with_escalation, transparent dissent, calibration anchors, split-aware individual-versus-panel metrics, correlation diagnostics, invariance diagnostics, and deterministic JSON and Markdown reports.
- **Type:** code
- **Issue:** #
- **Files:** jurys_out/aggregation.py, jurys_out/analysis.py, jurys_out/report.py, tests/test_aggregation.py, tests/test_analysis.py, tests/test_report.py, tests/anchors/, tests/fixtures/invariance/
- **Flags:** --reviewers code --isolation worktree
- **Produces:** A report that preserves every vote/evidence reference, distinguishes insufficient data, and compares a panel with its best individual judge.
- **Done when:** Frozen fixtures prove score(good) > score(garbage), holdout is the only selection-claim split, order/rename/comment variants retain adjudication, tie/abstention escalates, and the report is byte-stable for the same events.
- **Depends on:** 3

### Step 5: OpenAI Responses adapter
- **Status:** TODO
- **Problem:** Add a fail-closed OpenAI adapter that loads only OPENAI_API_KEY, requests structured output with store=false, validates response status and schema locally, captures usage, and redacts secrets from errors.
- **Type:** code
- **Issue:** #
- **Files:** jurys_out/adapters/openai.py, jurys_out/runner.py, docs/judge-card-schema.md, tests/test_openai_adapter.py
- **Flags:** --reviewers deep --isolation worktree
- **Produces:** An OpenAI Judge Card can issue one bounded structured judgment or return a typed unavailable/incomplete result without a network call in tests.
- **Done when:** Mock-transport tests cover success, rate limit, malformed body, incomplete response, and missing key; no test fixture contains a real key.
- **Depends on:** 3

### Step 6: Anthropic Messages adapter
- **Status:** TODO
- **Problem:** Add a fail-closed Anthropic adapter that loads only ANTHROPIC_API_KEY, requests output_config.format JSON Schema, validates response status and schema locally, captures usage, and redacts secrets from errors.
- **Type:** code
- **Issue:** #
- **Files:** jurys_out/adapters/anthropic.py, jurys_out/runner.py, docs/judge-card-schema.md, tests/test_anthropic_adapter.py
- **Flags:** --reviewers deep --isolation worktree
- **Produces:** An Anthropic Judge Card can issue one bounded structured judgment or return a typed unavailable/incomplete result without a network call in tests.
- **Done when:** Mock-transport tests cover success, rate limit, refusal, malformed body, and missing key; no test fixture contains a real key.
- **Depends on:** 3

### Step 7: Transparency surface and release-ready examples
- **Status:** TODO
- **Problem:** Finish the public CLI documentation, user-facing Judge/Panel Card explanation, security and retention guidance, example panel, smoke suite, and report walkthrough.
- **Type:** code
- **Issue:** #
- **Files:** README.md, docs/methodology.md, docs/judge-card-schema.md, judges/, panels/, suites/, tests/test_docs_examples.py
- **Flags:** --reviewers code --isolation worktree
- **Produces:** A fresh user can validate the example panel, understand the aggregation and limits, and see why raw artifacts are local-only.
- **Done when:** Documentation command checks pass against the fake adapter and every report field maps to a documented card, case, or event field.
- **Depends on:** 4, 5, 6

### Manual Steps

### Step M1: Budget-limited provider substrate smoke
- **Source step:** Step 7
- **Issue:** #
- **Commands:**
  ~~~~powershell
  cd C:\Users\abero\dev\jurys-out
  $env:OPENAI_API_KEY = "<set only in this shell>"
  uv run jurys-out smoke --provider openai --max-requests 1 --max-input-bytes 12000 --max-output-tokens 300 --max-estimated-cost-usd 1
  $env:ANTHROPIC_API_KEY = "<set only in this shell>"
  uv run jurys-out smoke --provider anthropic --max-requests 1 --max-input-bytes 12000 --max-output-tokens 300 --max-estimated-cost-usd 1
  ~~~~
- **What to look for:**
  | Check | Expected outcome |
  |---|---|
  | Credential handling | The CLI sees only the environment value and never prints it. |
  | Provider contract | Each configured provider returns one schema-valid structured result or a typed, redacted failure. |
  | Cost containment | Each invocation stays within the stated one-request, token, and $1 estimated-cost ceilings. |
  | Artifact retention | The run manifest and events exist; raw output is local and ignored by Git. |

### Step M2: Seed-panel benchmark and transparency review
- **Source step:** Step 7
- **Issue:** #
- **Commands:**
  ~~~~powershell
  cd C:\Users\abero\dev\jurys-out
  uv run jurys-out run --panel panels/example-code-review.toml --suite suites/smoke.jsonl --max-requests 8 --max-input-bytes 20000 --max-output-tokens 600 --max-estimated-cost-usd 5
  uv run jurys-out report --run <run_id printed by the prior command>
  ~~~~
- **What to look for:**
  | Check | Expected outcome |
  |---|---|
  | Completion policy | A missing required judge marks the run incomplete; no substitute model or fabricated panel score appears. |
  | Transparency | The report exposes configuration hashes, per-judge votes, evidence references, abstentions, dissent, and aggregation. |
  | Measurement claims | Small-sample or ambiguous results are labeled as such; correlation is not reported as zero when unavailable. |
  | Best-single comparison | A completed result compares the panel with each constituent judge under the same suite and limits. |

Please run M1 next after all automated steps complete.

## 13. Appendix

### Decision Inventory

| ID | P/D | Choice | Status |
|---|---|---|---|
| P1 | P | Public project with the tagline "Codifying judge panels for agentic review workflows, because the jury is still out." | current |
| P2 | P | Manage transparent judge panels for agentic development review. | current |
| P3 | P | Version 1 includes a simple benchmark runner; panel size is empirical, not fixed at five. | current |
| P4 | P | Local CLI-first release. | current |
| P5 | P | Benchmark cases contain task/specification, diff, test or CI evidence, candidate finding, and human adjudication. | current |
| P6 | P | Roles are not forced to be MECE; seek less redundant failure overlap and transparent judges. | current |
| D1 | D | Python >=3.12, uv, hatchling, argparse, TOML, and JSONL form the thin local stack. | current |
| D2 | D | OpenAI Responses and Anthropic Messages are the first two direct provider adapters. | current |
| D3 | D | Full raw provider output is retained locally by default, Git-ignored, and excluded from public reports. | current |
| D4 | D | A foreground append-only runner resumes only exact matching frozen inputs. | current |
| D5 | D | Required-member completion plus majority_with_escalation is the version-1 aggregation rule. | current |
| D6 | D | Request, input-byte, and output-token ceilings are always enforced; USD estimates are optional and versioned. | current |
| D7 | D | Calibration anchors, frozen holdouts, and insufficient-data reporting gate measurement claims. | current |
| D8 | D | Seven automated build slices use worktrees; provider/persistence seams use deep review and live checks are manual. | current |
| D9 | D | MIT license for the public release. | current |
| D10 | D | A versioned local TOML price book is required for an estimated-USD ceiling; M1/M2 caps are $1/$5. | current |
| D11 | D | Every case declares a split; order-swap, identifier-renaming, and misleading-comment invariance fixtures are required. | current |
| D12 | D | Retry timeout, connection, HTTP 429, and HTTP 5xx failures at most twice, respect Retry-After, persist every attempt, and never retry auth, validation, refusal, or other HTTP 4xx failures. | current |
| D13 | D | Retain Python >=3.12 and use a standard-library UUIDv4 run suffix. | current |

### finding-v1 structured result

~~~~json
{
  "schema_version": 1,
  "case_id": "case-sha256-<64 lowercase hex>",
  "finding_id": "finding-sha256-<64 lowercase hex>",
  "verdict": "valid | invalid | abstain",
  "confidence": 0.0,
  "evidence": [
    {
      "artifact": "diff | task | ci_log",
      "location": "stable line, hunk, or test identifier",
      "quote": "short supporting excerpt"
    }
  ],
  "rationale": "concise user-visible explanation",
  "abstain_reason": "required only when verdict is abstain"
}
~~~~

### Required example Panel Card fields

~~~~toml
schema_version = 1
panel_id = "example-code-review"
members = ["evidence-skeptic", "regression-hunter", "spec-reader"]
all_members_must_respond = true

[aggregation]
kind = "majority_with_escalation"
tie_verdict = "escalate"
all_abstain_verdict = "escalate"
~~~~

### Evidence basis

- Diverse panels can sometimes outperform a single large judge, but their value is
  contingent on genuine diversity rather than panel size alone:
  [Replacing Judges with Juries](https://arxiv.org/abs/2404.18796).
- Correlated model errors can make a large nominal panel behave like far fewer
  independent votes; this is a relevant but not yet peer-reviewed caution for
  code review: [Nine Judges, Two Effective Votes](https://arxiv.org/abs/2605.29800).
- LLM judges have demonstrated order-related bias, so invariance and calibrated
  examples are required: [Large Language Models are not Fair Evaluators](https://aclanthology.org/2024.acl-long.511/).
- Code-specific judge benchmarks report sensitivity to ordering, variable naming,
  and misleading comments: [CodeJudgeBench](https://aclanthology.org/2026.acl-long.888/).
