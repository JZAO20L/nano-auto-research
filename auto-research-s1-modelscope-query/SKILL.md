---
name: auto-research-s1-modelscope-query
description: Query ModelScope for models and datasets. Use exact lookup API (search API unreliable via curl). Produces entries for assets.md with download commands.
metadata:
  version: "0.1"
---

# S1 ModelScope Query

Verify model/dataset availability on ModelScope and prepare download commands.

## 1. Exact Model Lookup

```bash
curl -s --max-time 20 "https://modelscope.cn/api/v1/models/NAMESPACE/MODEL_NAME"
```

Example:
```bash
curl -s --max-time 20 "https://modelscope.cn/api/v1/models/Qwen/Qwen2.5-7B-Instruct"
```

Check response:
- `Code` == 200 → model exists
- `Data.Name` — confirmed model name
- `Data.Downloads` — download count (popularity signal)
- `Data.UpdatedAt` — last update

If `Code` != 200, the model does not exist under that namespace/name.

## 2. Exact Dataset Lookup

```bash
curl -s --max-time 20 "https://modelscope.cn/api/v1/datasets/NAMESPACE/DATASET_NAME"
```

Example:
```bash
curl -s --max-time 20 "https://modelscope.cn/api/v1/datasets/modelscope/AdvBench"
```

Same response structure: check `Code` == 200 for existence.

## 3. Search Limitation

**ModelScope's search API is not reliable via curl.** Do not attempt:
```bash
# UNRELIABLE — do not use
curl -s --max-time 20 "https://modelscope.cn/api/v1/models?Query=keyword"
```

**Strategy**: Search on HuggingFace first (`auto-research-s1-huggingface-query`), then verify the same model exists on ModelScope using exact lookup with the expected namespace/name.

If HuggingFace is unavailable or the model is not found there, search ModelScope directly via the web UI or CLI: `modelscope list --model_name KEYWORD`.

Common namespace mappings:
- HuggingFace `Qwen/Qwen2.5-7B` → ModelScope `Qwen/Qwen2.5-7B`
- HuggingFace `meta-llama/Llama-3-8B` → ModelScope `LLM-Research/Meta-Llama-3-8B`

## 4. Download Commands

### CLI (preferred):
```bash
modelscope download --model 'Qwen/Qwen2.5-7B-Instruct' --local_dir './assets/models/Qwen2.5-7B-Instruct'
```

For datasets:
```bash
modelscope download --dataset 'modelscope/AdvBench' --local_dir './assets/data/AdvBench'
```

### Python:
```python
from modelscope.hub.snapshot_download import snapshot_download
snapshot_download('Qwen/Qwen2.5-7B-Instruct', cache_dir='./assets/models')
```

## 5. Output Format

Append to `docs/assets.md`:

```markdown
| Asset | Type | Platform | ID | Downloads | Download Command | Status |
|-------|------|----------|----|-----------|-----------------|--------|
| Qwen2.5-7B-Instruct | model | ModelScope | Qwen/Qwen2.5-7B-Instruct | 1.2M | `modelscope download --model 'Qwen/Qwen2.5-7B-Instruct' --local_dir './assets/models/Qwen2.5-7B-Instruct'` | pending |
```

## 6. Verification Checklist

Before marking an asset as "available":
- [ ] Exact lookup returns Code=200
- [ ] Model/dataset is publicly accessible (no gated access)
- [ ] Download command tested (at minimum, verify it starts without auth error)
