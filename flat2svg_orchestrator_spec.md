# flat2svg Orchestrator for Google Colab — Development Spec

## How to work with me
- Build in the stages below, one stage at a time. Before writing code for a stage, summarize your plan in a few lines and wait for my "yes".
- Tokens cost me a lot: be concise, don't rewrite whole files for small changes, don't run test suites unless I ask.
- All code, comments, docstrings, console messages, prompts and file contents in English. Chat with me in Russian.
- First ask me for the current `flat2svg.py` (or the repo URL) and read it before designing anything. Do not change its algorithm until stage 5.

## Goal
A Colab notebook that runs controlled improvement cycles on my existing `flat2svg.py` (batch flat PNG → SVG vectorizer). Three Claude models play fixed roles, and I approve at checkpoints:
- **Opus (lead engineer)** — reads the series report, proposes one hypothesis and a change plan. Called rarely, once per series.
- **Sonnet (executor)** — implements the approved plan as a patch to flat2svg only.
- **Haiku (tester)** — reads test/metric results produced by plain code and returns a short verdict: pass / regression / failure + reason.

Plain Python does everything that doesn't need an LLM: file copying, running conversions, rendering, metrics, JSON.

## Environment and data rules
- Google Colab, CPU only (no GPU unless explicitly needed).
- Google Drive is the source of truth. Mount it; every step writes its results and state to Drive immediately, so a session reset loses nothing. Never keep results only in `/content`.
- Input images live in nested subfolders: process recursively and mirror the same folder structure in all outputs.
- API key from Colab Secrets (`ANTHROPIC_API_KEY`), never in the notebook or Git.
- Model IDs in one config cell: `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`.
- UI: ipywidgets. No dropdown selects anywhere — use tabs/toggle buttons.

## Drive layout
```
Orchestrator/
  config.json            # paths, model IDs, budgets, retry limits
  golden_set/            # fixed 8–12 test images (nested subfolders allowed)
  runs/<run_id>/
    svg/  render/        # SVG output and SVG rendered back to PNG, mirrored structure
    metrics.jsonl
  code_versions/<version_id>/flat2svg.py
  log.jsonl              # one line per event, appended immediately
  state.json             # current step of the cycle, for resume
```

## The cycle
1. **Baseline** — run current flat2svg on golden_set, render SVG → PNG, compute metrics.
2. **Review** — gallery: source | baseline | variant at identical scale, with zoom on a crop. I mark best/reject per image and add a comment. Saved to Drive.
3. **Opus plan** — input: compact series report (aggregated metrics, worst cases, my comments, current code). Output: JSON `{hypothesis, changes, expected_effect, risks, test_criteria}`. **Stop: I approve, edit or reject.**
4. **Sonnet patch** — input: approved plan + current code. Output: full new `flat2svg.py` saved as a new code version (never overwrite the old one).
5. **Test** — plain code runs: syntax/import check, run on golden_set, SVG validity, metrics vs baseline.
6. **Haiku verdict** — input: test output + metric diff. Output: JSON `{verdict, reason, fix_hint}`. On failure → back to Sonnet with the hint, max 2 retries, then stop and ask me.
7. **Review again** — gallery baseline vs new version. **Stop: I accept or reject.** Accepted version becomes the new baseline.

## Metrics (per image, plain code)
SSIM and mean color difference (ΔE) between source and rendered SVG, number of paths and nodes, SVG file size, conversion time, valid/invalid SVG. Metrics are a filter; my visual judgment decides.

## Budget and safety
- Per-series and per-role budget limits in config (USD and tokens). On exceeding: stop and ask — never silently switch models.
- Log every API call to `log.jsonl`: role, model, input/output tokens, estimated cost, result status.
- Cache LLM answers by hash of the input.
- Sonnet may modify only `flat2svg.py`. No LLM-suggested shell commands are executed.
- Max retries and timeouts in config.

## Stages
- **Stage 0** — read my flat2svg, list its inputs, parameters, dependencies, outputs. Propose the config.json shape.
- **Stage 1** — notebook skeleton: Drive mount, setup, config, recursive batch run, SVG render, metrics, log.jsonl, state.json resume.
- **Stage 2** — review gallery with tabs, marks and comments saved to Drive.
- **Stage 3** — Haiku tester step + budget/logging wrapper for API calls.
- **Stage 4** — Sonnet executor step with versioned code and retry loop.
- **Stage 5** — Opus lead-engineer step and the full cycle with both approval stops.

Done when: I can open the notebook after a session reset, it resumes from state.json, and a full cycle runs with only my two approvals as manual steps.
