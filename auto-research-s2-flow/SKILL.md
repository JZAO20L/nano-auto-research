---
name: auto-research-s2-flow
description: Stage 2 flow control — orchestrates coding & experimenting from confirmed idea to paper-ready results
metadata:
  version: "0.1"
---

# Stage 2: Coding & Experimenting — Flow Control

## Goal

From a confirmed research idea + prepared assets → produce:
- Experiment repository with reproducible code (src/, scripts/, exp/)
- `docs/experiment_results.md` with all key results
- `docs/pre_review_checklist.md` with experiment matrix

## Sub-task Sequence

Execute in order. After each step, update `docs/stage2_progress.md`.

### Step 0: Model Verification

Read `project_config.yaml` from the project root. For each model entry in the `models:` list:

**Local models** (`type: local`):
1. Check if the model exists at the specified `path` (look for `config.json` or `*.safetensors`)
2. If missing: invoke `auto-research-s2-asset-download` to download the model to the specified path
3. After download, verify the path now contains model files

**API models** (`type: api`):
1. Invoke `auto-research-s2-model-call` to send a test request (e.g., a simple "Hello" prompt with max_tokens=5)
2. If the request fails: report the error and ask the user to check the URL/API key
3. Do NOT proceed with experiments until all API models respond successfully

Only proceed to Step 1 after ALL models in the config are verified.

### 1. Verify Assets Ready

Check that required assets exist:
- `data/` — datasets populated and loadable
- `models/` — model weights downloaded (or remote API configured)
- `baselines/` — baseline repos cloned with dependencies installable
- `project_config.yaml` — user-provided URLs/keys present

If missing → invoke **auto-research-s2-asset-download**.

### 2. Build Eval Infrastructure

Invoke **auto-research-s2-eval-infrastructure** to create:
- `scripts/eval.py` — unified evaluation script
- Metric functions (ASR judge, keyword match, etc.)
- Result I/O format (JSON per-sample + aggregated)

### 3. Sanity Check: Reproduce Baseline

Run the primary baseline on a small subset (50–100 samples).
Compare with reported paper numbers. Acceptable tolerance: **±2%**.

- If within tolerance → proceed.
- If outside → debug (data version, hyperparams, prompt format). Do NOT proceed until resolved.

### 4. Pilot Experiment

Run the proposed method on 100–200 samples through the full pipeline:
- Data loading → method execution → evaluation → result saving

Goal: catch integration bugs before committing GPU-hours.

### 5. Implement Baselines

For each baseline method:
- Wrap in `src/baselines/NAME.py` (OOP interface)
- Create `exp/baseline_NAME.sh`
- Verify it reproduces reported numbers

Reference: **auto-research-s2-model-call** for API patterns, **auto-research-s2-model-training** for training loops. For local model serving, reference **auto-research-s2-vllm-deploy** to set up vLLM servers before calling models.

### 6. Implement Core Method

Implement the proposed method in `src/methods/`. Key requirements:
- Clean OOP interface with `run()` method
- Configurable via YAML or CLI args
- Logging at key steps

### 7. Generate Pre-review Checklist

Create `docs/pre_review_checklist.md`:
- Experiment matrix: method × dataset × metric
- Mark experiments as **must-run** or **nice-to-have**
- Ablation list: which components to remove/replace
- Efficiency experiments: wall-clock time, GPU memory

### 8. Run Experiments

Invoke **auto-research-s2-experiment-runner** to execute the matrix.

### 9. Analyze Results

Invoke **auto-research-s2-result-analysis** to produce:
- Markdown tables (main results, ablation, efficiency)
- Key findings as bullet points
- Failure analysis section

## Research Type Branch

| Type | 做题型 (Optimize on Benchmarks) | 出题型 (Create Benchmark) |
|------|------|------|
| Focus | Beat SOTA on existing datasets | Construct dataset + demonstrate gap |
| Eval | Standard metrics vs baselines | Quality/diversity metrics + baseline performance |
| Extra work | — | Data construction pipeline, annotation, release |

Detect type from the idea document. Adjust step 7 checklist accordingly.

## Rollback Protocol

- **Single idea fails** (pilot shows no signal after debugging): move to next idea in the idea pool. Log failure reason in `docs/stage2_progress.md`.
- **All ideas in pool fail**: report back to auto-research orchestrator for Stage 1 rollback (re-generate ideas).

## Decision Gate

After step 9, present to user:
1. Summary table of main results
2. Whether results are sufficient for a paper claim
3. Gaps or weaknesses identified

Wait for user confirmation before marking Stage 2 complete.

## Progress Tracking

Maintain `docs/stage2_progress.md` with:
```markdown
## Asset Checklist
- [x] Dataset X downloaded
- [ ] Model Y downloading...

## Current Step: 4 (Pilot)
## Blockers: none
## Ideas Tried: 2/5
```
