---
name: auto-research-s2-result-analysis
description: Guide for analyzing experiment results — tables, statistical checks, pattern identification, failure analysis
metadata:
  version: "0.1"
---

# Result Analysis Guide

## Input

Read all results from `output/` subdirectories:
```python
import json
from pathlib import Path

def load_all_results() -> list[dict]:
    results = []
    for f in sorted(Path("output").rglob("results.json")):
        data = json.loads(f.read_text())
        data["_path"] = str(f)
        results.append(data)
    return results
```

## Generate Markdown Tables

### Main results table

```markdown
| Method | AdvBench | HarmBench | JailbreakBench | Avg |
|--------|----------|-----------|----------------|-----|
| GCG    | 45.2%    | 38.1%     | 41.0%          | 41.4% |
| PAIR   | 62.3%    | 55.7%     | 58.2%          | 58.7% |
| **Ours** | **85.1%** | **79.3%** | **82.0%**   | **82.1%** |
```

Rules:
- Bold the best result per column
- Include ±std if multi-seed: `85.1% ± 1.2`
- Last column = average across datasets

### Ablation table

```markdown
| Variant | ASR | Δ vs Full |
|---------|-----|-----------|
| Full method | 85.1% | — |
| w/o skill evolution | 71.3% | -13.8 |
| w/o self-reflection | 78.9% | -6.2 |
| w/o diversity | 80.2% | -4.9 |
```

### Efficiency table

```markdown
| Method | Time/sample (s) | GPU Mem (GB) | Total queries |
|--------|----------------|--------------|---------------|
| GCG    | 45.2           | 24.1         | 500           |
| Ours   | 3.8            | 16.2         | 12            |
```

## Statistical Checks

- If improvement over best baseline **< 5% absolute**: flag as "needs multi-seed verification"
- Run 3 seeds minimum for claims with < 5% margin
- Report: `mean ± std` across seeds
- If std > improvement: result is NOT significant — note this explicitly

```markdown
⚠️ Improvement of 3.2% over PAIR — below 5% threshold.
Multi-seed (n=3): 85.1% ± 2.8% — high variance, needs more seeds or larger dataset.
```

## Comparison with Baselines

For each metric, compute:
- **Absolute Δ**: `ours - baseline` (e.g., +22.8%)
- **Relative improvement**: `(ours - baseline) / baseline × 100` (e.g., +36.6%)

```python
def compute_delta(ours: float, baseline: float) -> dict:
    if baseline == 0:
        return {"absolute": ours - baseline, "relative": "N/A"}  # undefined when baseline is 0
    return {
        "absolute": ours - baseline,
        "relative": (ours - baseline) / baseline * 100
    }
```

## Pattern Identification

Analyze per-sample results to find:

1. **Which categories benefit most**: group by category/tag, compute per-group ASR
2. **Which fail**: samples where method fails but baseline succeeds (regression)
3. **Difficulty correlation**: does success rate drop with prompt complexity/length?

```python
from collections import defaultdict

def analyze_by_category(per_sample: list[dict]) -> dict:
    groups = defaultdict(list)
    for s in per_sample:
        cat = s.get("category", "unknown")
        groups[cat].append(s["success"])
    return {cat: sum(v)/len(v) for cat, v in groups.items()}
```

## Failure Analysis

Document in a dedicated section:

```markdown
## Failure Analysis

### Cases where method fails
- Category X: 40% ASR (vs 85% overall) — hypothesize: prompts too short for skill matching
- Model Y: 60% ASR (vs 82% on Model Z) — hypothesize: stronger safety training

### What didn't work
- Variant A (no temperature scaling): degraded by 12% — diversity matters
- Prompt format B: caused parsing failures in 15% of cases

### Hypotheses for future work
- Failure on category X may be addressed by category-specific skills
- Stronger models may need multi-turn approach
```

## Output

Update `docs/experiment_results.md` with this structure:

```markdown
# Experiment Results

## Main Results
[table]

## Ablation Study
[table]

## Efficiency
[table]

## Key Findings
- Finding 1: ...
- Finding 2: ...

## Failure Analysis
[analysis]

## Statistical Notes
- [any flags about significance]
```

## Format Rules

- All tables in **markdown** (convertible to LaTeX later)
- Percentages with 1 decimal: `85.1%`
- Bold best results in each column
- Key findings as bullet points (max 5–7)
- Every claim backed by a number from the tables
