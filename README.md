# Replicating reward-hacking probes on open models (CS2881R mini-project)

Replication + extension of Goodfire's *"Monitoring and Discovering Reward Hacking with Internal
Representations during LLM Evaluations"* (arXiv 2609.19101) on small open models (OLMo 3.1 32B Think,
later a second model), instead of the paper's 1T+ frontier models.

## Status
- **Phase 0 — P(reward hack): DONE.** Measured how often OLMo 3.1 32B Think reward-hacks on
  Impossible-LiveCodeBench. Result: it is a *competent naive hacker* — ~46% hack propensity on impossible
  tasks (near the frontier band), ~29% successful hacks where hardcoding suffices, but ~0% where the hack
  needs sophistication it lacks. See `results/exp1/FINAL_RESULTS.md`.
- **Phase 1 — DoM probe replication: IN PROGRESS.** Build a difference-of-means reward-hacking direction
  from synthetic contrast pairs, show it detects hacking in the Phase-0 rollouts (vs an LLM monitor),
  read it out with the logit lens, and test it causally with honeypot steering. Plan in
  `notes/replication_plan.md`.

## Layout
- `notes/` — `paper_notes.md` (method we replicate), `background_research.md` (models/benchmarks/prior work),
  `replication_plan.md` (Phase-1 plan), `exp1_impossible_lcb_protocol.md` (Phase-0 methods + findings).
- `exp1/` — Phase-0 code: run rollouts (`run_pilot.py`), CoT hack judge (`judge_cot.py`,
  `judge_consensus.py`), analysis (`analyze.py`), serving (`vm_serve_and_harness.sh`, `smoke_test.py`).
- `synth/` — `generate_pairs.py`: Claude-generated hack/honest minimal pairs for building DoM vectors.
- `results/exp1/` — `transcripts_full.jsonl` (150 rollouts), `labels_consensus.jsonl` (3× judge labels),
  `FINAL_RESULTS.md`.
- `patches/` — audit fixes for ImpossibleBench (exit-code & find_code bugs, `exclude_broken`).

## Setup
Rollouts use [ImpossibleBench](https://github.com/safety-research/impossiblebench) (clone into `vendor/`,
apply `patches/impossiblebench-lcb-fixes.patch`) served against a local vLLM OLMo endpoint. The judge and
synthetic-data generator use the Anthropic API. See `notes/` for specifics.
