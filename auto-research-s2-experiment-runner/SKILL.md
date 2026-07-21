---
name: auto-research-s2-experiment-runner
description: Guide for orchestrating experiment execution — script conventions, GPU management, result collection
metadata:
  version: "0.1"
---

# Experiment Runner Guide

## Convention

**One file = one complete experiment.** Each experiment lives in `exp/EXP_NAME.sh`.

## Script Template

```bash
#!/bin/bash
# Experiment: EXP_NAME
# Description: One-line description of what this tests
# Expected output: output/EXP_NAME/results.json
set -e

EXP_NAME="EXP_NAME"
OUTPUT_DIR="output/${EXP_NAME}"
mkdir -p "${OUTPUT_DIR}"

echo "[$(date)] Starting ${EXP_NAME}"

# 1. Start models (if needed)
# CUDA_VISIBLE_DEVICES=0 python -m vllm.entrypoints.openai.api_server \
#   --model ./models/qwen3-4b --port 8000 &
# sleep 30  # wait for server ready

# 2. Run experiment
python scripts/eval.py \
  --dataset advbench \
  --method "${EXP_NAME}" \
  --output-dir output \
  --mode local

# 3. Collect results
echo "[$(date)] ${EXP_NAME} complete"
echo "Results: ${OUTPUT_DIR}/results.json"

# 4. Cleanup (kill model servers if started)
# kill %1 2>/dev/null || true
```

Make executable: `chmod +x exp/EXP_NAME.sh`

## Multi-Experiment Management

Run experiments sequentially to avoid GPU contention:

```bash
# Run all experiments in order
for exp in exp/*.sh; do
  echo "=== Running: $exp ==="
  bash "$exp" 2>&1 | tee "output/$(basename $exp .sh)/log.txt"
  echo "=== Done: $exp (exit: $?) ==="
done
```

Log stdout+stderr: `bash exp/NAME.sh 2>&1 | tee output/NAME/log.txt`

## GPU Management

### Before starting an experiment

```bash
# Check GPU availability
nvidia-smi --query-gpu=index,memory.used,memory.total,utilization.gpu --format=csv

# Kill stale vLLM/python processes on target GPU
fuser -k /dev/nvidia0 2>/dev/null || true
# Or by pattern:
pkill -f "vllm.*port 8000" 2>/dev/null || true
```

### GPU assignment convention

| GPU | Role |
|-----|------|
| 0 | Policy model / training |
| 1 | Target model / evaluation |
| 2+ | Parallel experiments or guard model |

### Memory check

```python
import torch
free_mem = torch.cuda.mem_get_info(device=0)
print(f"GPU 0: {free_mem[0]/1e9:.1f}GB free / {free_mem[1]/1e9:.1f}GB total")
```

## Result Collection

After each experiment completes, extract key metrics and append to `docs/experiment_results.md`:

```python
# scripts/collect_results.py
import json
from pathlib import Path

def collect_all():
    lines = ["# Experiment Results\n"]
    for results_file in sorted(Path("output").rglob("results.json")):
        data = json.loads(results_file.read_text())
        method = data.get("method", results_file.parent.name)
        dataset = data.get("dataset", results_file.parent.parent.name)
        metrics = data.get("metrics", {})
        asr = metrics.get("asr", "N/A")
        lines.append(f"| {method} | {dataset} | {asr:.2%} |" if isinstance(asr, float) else f"| {method} | {dataset} | {asr} |")
    return "\n".join(lines)
```

## Experiment Matrix

From `docs/pre_review_checklist.md`, categorize experiments:

| Priority | Criteria | Example |
|----------|----------|---------|
| **Must-run** | Core claim depends on it | Main method vs baselines on primary dataset |
| **Must-run** | Reviewer will ask | Ablation of key component |
| **Nice-to-have** | Strengthens but not essential | Extra dataset, efficiency analysis |

Run must-run first. If time/compute limited, skip nice-to-have.

## Failure Handling

If an experiment fails:
1. Log the error in `output/EXP_NAME/log.txt`
2. Mark status in `docs/stage2_progress.md`:
   ```markdown
   - [x] exp/main_results.sh — SUCCESS
   - [ ] exp/ablation_no_skill.sh — FAILED: OOM on GPU 0
   ```
3. Diagnose: OOM → reduce batch size; timeout → increase limit; import error → fix deps
4. Continue with next experiment — do NOT block the pipeline
5. Retry failed experiments after fixing the issue

## Naming Convention

```
exp/main_METHOD_DATASET.sh      # Main comparison
exp/ablation_COMPONENT.sh       # Ablation study
exp/efficiency_METHOD.sh        # Time/memory measurement
exp/transfer_SRC_TGT.sh         # Transfer experiment
```
