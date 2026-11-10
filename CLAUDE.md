# CLAUDE.md: Deckard build guide

This file tells Claude (Claude Code) how to build this project. Read **README.md** first: it is the design spec. This file is the build plan. **INFO.md** is the interview prep companion; keep it in sync when a design decision changes.

## Goal

Regression testing for LLM apps. Point Deckard at any LLM app (HTTP endpoint or provider SDK) plus a YAML test suite, and it runs the cases, scores them, stores the results, and reports regressions against a baseline. Scorers: exact/regex/JSON Schema checks, LLM-as-a-judge with a rubric and calibration against a small human-labeled set (report judge-human agreement as Cohen's kappa), RAG metrics (faithfulness, answer relevance, context precision, context recall, RAGAS-style), and a red-team pack (prompt injection, jailbreak, PII leakage). A GitHub Action runs a suite on a PR and fails it when scores drop past a threshold. A small dashboard shows runs, diffs and cost. Case study: evaluating `striderzz/RAG_Project1` before and after a change.

Scope is sized for one person in 2 to 4 weeks. Anything not listed here goes under Stretch goals, not into the core.

## Tech stack

- Python 3.11, uv for dependencies, ruff + black, type hints
- Typer CLI: `deckard run`, `deckard compare`, plus `calibrate`, `baseline save` and `serve`
- Pydantic v2 for suites, results and judge output; PyYAML (`safe_load`); jsonschema
- httpx (async) for HTTP targets
- Official `anthropic` and `openai` SDKs (no multi-provider wrapper)
- Judge models: `claude-haiku-5-5` by default, `claude-opus-5-5` only for calibration and for re-judging borderline or pass-to-fail cases
- SQLite via stdlib `sqlite3` in WAL mode
- FastAPI read-only API for the dashboard
- Next.js (App Router) + TypeScript + Tailwind + lucide icons for the dashboard
- pytest + pytest-asyncio, LLM calls mocked in tests
- GitHub Actions: `ci.yml` (lint, tests, dashboard build) and `eval.yml` (PR gate via the composite action in `action/`)
- Docker Compose + Makefile (`make up`) for the API and dashboard

## Build order (phases)

1. Package scaffold, Pydantic suite schema, `deckard run`, target adapters (HTTP, Anthropic, OpenAI), deterministic scorers (exact, regex, JSON Schema), SQLite store, pytest, `ci.yml`.
2. LLM judge: rubrics in `rubrics/`, structured output through a forced tool call, judge cache keyed on a hash of everything that affects the verdict, token and USD logging per call (`pricing.yaml`), per-run budget cap, resume.
3. Calibration: hand-labeled set in `calibration/labels.yaml`, `deckard calibrate`, Cohen's kappa with bootstrap CI for Haiku and Opus, confusion matrix, Opus escalation for boundary scores.
4. RAG metrics: faithfulness, answer relevance (OpenAI embeddings, judge fallback), context precision, context recall.
5. Red-team pack in `suites/redteam/`: prompt injection (direct and indirect, canary strings), jailbreak (judge rubric, app-policy probes only), PII leakage (seeded synthetic PII, regex + Luhn).
6. `deckard compare` (per-scorer deltas, bootstrap CI, new failures, Opus re-judge of flips, Markdown output, exit code), committed baselines in `baselines/` via `deckard baseline save`, composite GitHub Action and `eval.yml`.
7. FastAPI read API and Next.js dashboard: runs list, run detail, compare, calibration, cost. Run `/unslop` before calling it done.
8. Case study: adapter in `examples/rag_project1/`, suite with reference answers plus the red-team pack, runs before and after one change, write-up and real numbers in README.

## Stretch goals

Do not start these until Phase 8 is done.

- Pairwise A/B judging with order swapping
- OpenTelemetry trace import to build cases from real traffic
- Gemini and Ollama targets
- Hosted read-only dashboard demo on Vercel
- Synthetic case generation from seed cases
- Cross-check of the RAG metrics against the ragas library on a sample

## Working rules for Claude

- Build **one phase at a time**, in the order above. Each phase must run end to end before the next starts.
- After each phase, update the **Roadmap checklist in README.md** and commit with a clear message.
- **Never invent results.** Every number in README.md and INFO.md (kappa, scores, cost, run time, latency) must come from a run in this repo. Put the raw output in `docs/results/`. Until then, leave the placeholder `—`.
- Keep cost low: `claude-haiku-5-5` is the default judge, `claude-opus-5-5` only for calibration and escalation, cache judge calls, keep eval sets small, log tokens and USD for every call, and pass a budget cap on every real run. The total API budget is about $100.
- Tests never call real LLM APIs. Mock the clients.
- Keep secrets out of git: use `.env` (gitignored) plus a committed `.env.example`.
- Never commit raw datasets or large model files. Write download scripts and add the data folders to `.gitignore`.
- Python: 3.11+, `uv` for dependencies, `ruff` + `black` for formatting, type hints, `pytest` for tests.
- Every service gets a `Dockerfile`. Local development runs with `docker compose up` (`make up`).
- Add a GitHub Actions workflow early (lint + tests) and keep it green.
- Prefer small, readable modules over clever abstractions. Add docstrings where intent is not obvious.
- The UI follows the unslop skill (`~/.claude/skills/unslop`): run `/unslop` on the main screens before calling the UI done.
- **Screenshots:** when a phase has something visible (dashboard, CLI output, PR comment, test report), capture real screenshots into `docs/screenshots/` and embed them in README.md. Never mock them up.
- When a phase is done, add a short "How it works" note or diagram to `docs/`. This is what interviewers will read.

## Definition of done (whole project)

- `make up` (or the documented command) brings up the API and dashboard in one step
- `deckard run` and `deckard compare` work on the example and case-study suites
- The GitHub Action fails a PR on a real regression in the case study
- README has a real architecture diagram, setup steps, screenshots and measured results, including judge-human kappa and cost per 100 cases
- CI is green, tests cover the core logic
- A 2 to 3 minute demo video or GIF is linked in the README

## Status

Only `README.md`, `INFO.md` and this file exist so far. Start at **Phase 1**.
