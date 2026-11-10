# CLAUDE.md: Assay build guide

This file tells Claude (Claude Code) how to build this project. Read **README.md** first: it is the design spec. This file is the build plan.

## Goal

Evaluation and guardrails platform for LLM apps: YAML test suites, deterministic + RAG + LLM-as-a-judge scorers with judge calibration, red-team probes, a results dashboard and a GitHub Action that gates PRs on quality, cost and latency.

## Tech stack

Python 3.11, asyncio + httpx, Pydantic, Typer CLI, LiteLLM (multi-provider), RAGAS, FastAPI, PostgreSQL, Redis, React + Vite + Tailwind, OpenTelemetry, Docker Compose, GitHub Actions.

## Build order (phases)

1. Scaffold `assay/` package, Typer CLI (`assay run`), YAML suite schema, example suite.
2. Async runner with rate limiting, retries and caching; provider adapters via LiteLLM + generic HTTP target.
3. Deterministic scorers (schema, regex, exact/fuzzy match) and RAG scorers.
4. LLM-as-a-judge rubrics + calibration against a small human-labelled set (Cohen's kappa).
5. Synthetic and red-team case generation (injection, jailbreak, PII leak).
6. Results API on PostgreSQL + React dashboard with run-to-run diffs.
7. GitHub Action that runs a suite and comments a score diff on the PR; fail thresholds.
8. Case study in `examples/`: a sample RAG app where Assay catches a real regression; numbers in README.

## Working rules for Claude

- Build **one phase at a time**, in the order below. Each phase must run end to end before the next starts.
- After each phase, update the **Roadmap checklist in README.md** and commit with a clear message.
- **Never invent results.** Every number in README.md (accuracy, latency, throughput, cost) must come from a run in this repo. Put benchmark output in `docs/results/`.
- Keep secrets out of git: use `.env` (gitignored) plus a committed `.env.example`.
- Never commit raw datasets or large model files. Write download scripts and add the data folders to `.gitignore`.
- Python: 3.11+, `uv` or `pip-tools` for dependencies, `ruff` + `black` for formatting, type hints, `pytest` for tests.
- Every service gets a `Dockerfile`. Local development runs with `docker compose up`.
- Add a GitHub Actions workflow early (lint + tests) and keep it green.
- Prefer small, readable modules over clever abstractions. Add docstrings where intent is not obvious.
- **Screenshots:** when a phase has something visible (UI, dashboard, Grafana, CLI output, test report), capture real screenshots into `docs/screenshots/` and embed them in README.md. Never mock them up.
- When a phase is done, add a short "How it works" note or diagram to `docs/`. This is what interviewers will read.

## Definition of done (whole project)

- `docker compose up` (or the documented command) brings up a working demo in one step
- README has a real architecture diagram, setup steps, screenshots or GIFs, and measured results
- CI is green, tests cover the core logic
- A 2-3 minute demo video or GIF is linked in the README

## Status

Only `README.md` and this file exist so far. Start at **Phase 1**.
