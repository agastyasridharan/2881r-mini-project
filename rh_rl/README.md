# RL against the reward-hacking probe (gpt-oss-120b)

Follow-up to `rh_probe/`. The question is whether a policy trained with the DoM probe in its reward stops hacking or
learns to evade the probe. The write-up is section T6 of `rh_probe/report/rh_probe_report_filled.html`.

## Setup
- **Algorithm.** GRPO on gpt-oss-120b (medium reasoning effort) in a single-turn Impossible-LiveCodeBench
  environment: 16 prompts × 8 samples per step, 100 steps.
- **Trainable weights.** LoRA (r = 32) on the attention projections only. The MXFP4 experts stay frozen in both the
  trainer and the vLLM samplers; see `notes/trainer_efficiency.md` §2.
- **Probe.** Pre-registered `span__all`, layer 13, max over reasoning tokens, with threshold τ = 770537 (10% FPR on
  the calibration pilot).
- **Arms.** All arms get the pass reward. They differ in how the probe enters:
  - A: no probe.
  - B: a −λ penalty (λ = 1) when the probe score exceeds τ.
  - C: a linear unclipped penalty, (score − τ)/448548.
  - D: probe-flagged samples are dropped (filter).
- **Evaluation.** 309 held-out Impossible-LCB tasks. The lenient prompt uses 8 samples per task, the strict prompt
  4. All samples are judged by GPT-5.6 Sol.

## Held-out results (impossible splits, lenient prompt)

| | base | A | B | C | D |
|---|---|---|---|---|---|
| Hack rate | 44% | 64% | 53% | 36% | 56% |
| Probe recall at τ | 69% | 78% | **29%** | 42% | 68% |
| Hacks the probe misses (share of samples) | 13% | 14% | **37%** | 21% | 18% |
| Probe FPR at τ | 14% | 25% | 6% | 5% | 16% |

- **B evades the fixed threshold, not the probe as a whole.** AUROC is essentially unchanged (base 0.88, A 0.85,
  B 0.87). B's hack scores shift to just under τ.
- **Most of B's evasion is in the text.** Cross-scoring attributes about 76–85% of the recall drop to what B
  writes rather than to changed weights (`results/score_analysis.json`).
- **A reasoning-only LLM monitor still catches B.** It flags 97% of B's hacks at 3% FPR (`results/cot_monitor/`).
- **C hacks least,** but its strict-prompt hacking rises from 6% (base) to 17%.
- **D keeps recall near base,** but only 22% of its samples carry gradient, which confounds the comparison.

## Layout
- **`cluster/`**: GPU-host code, the versions that actually ran.
  - Trainer: `rl_train_gptoss.py`. It imports `tests/dq_experts.py` (MXFP4 experts dequantised just in time).
  - vLLM loader for attention-only weight pushes: `vllm_ext.py`.
  - Samplers: `serve_*_rl.sh`, `sample.py`.
  - Arm and eval runners, and the watchdogs: `run_*.sh`, `watchdog*.sh`.
  - Probe scoring: `score_probe*.py`, `score_analyze.py`.
  - Tasks: `train_tasks.jsonl`, `ib_eval_tasks.jsonl`.
  - Pinned environment: `base_freeze.txt`.
  - Per-step training logs and arguments: `runs/arm*/{args.json,log.jsonl}`.
  - CPU-side feasibility checks from the design phase: `agent_scratch/`.
  - Earlier or alternative trainers: `rl_train.py` (Qwen), `rl_train_gptoss_v3.py`.
- **Top-level `*.py`**: local analysis.
  - Judging: `judge_rl.py`, `judge_eval.py`.
  - Stats: `eval_stats.py`, `c_stats.py`, `d_stats.py`.
  - Reasoning-only monitor: `cot_monitor.py`.
  - Lexical analysis: `lex_diff.py`, `peak_stats.py`.
  - Figures: `figs/make_figs.py`, with outputs in `figs/`.
  - Tasks: `gen/` builds them from LiveCodeBench.
- **`results/`**: GPT-5.6 Sol verdicts on every training sample (`judge_rl/`) and every held-out sample
  (`judge_eval/`), plus the monitor verdicts, the score traces and the score analysis.
  - The scripts read these from `data/` on the original machine.
  - The scripts that ssh to the GPU host take the target from `RL_HOST`.
- **`notes/`**: design notes on trainer and sampler efficiency.

Raw samples (~1.2 GB), checkpoints and the calibration pilot dumps are not included.
