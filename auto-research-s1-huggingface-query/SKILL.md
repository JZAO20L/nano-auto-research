---
name: auto-research-s1-huggingface-query
description: Query HuggingFace Hub API for models, datasets, and papers. Produces entries for assets.md with download commands.
metadata:
  version: "0.1"
---

# S1 HuggingFace Query

Search and verify models/datasets on HuggingFace Hub.

## 1. Model Search

```bash
curl -s --max-time 20 "https://huggingface.co/api/models?search=KEYWORD&limit=10&sort=downloads"
```

Example:
```bash
curl -s --max-time 20 "https://huggingface.co/api/models?search=Qwen2.5-7B&limit=10&sort=downloads"
```

Response is a JSON list. Each item:
- `id` — model identifier (e.g., "Qwen/Qwen2.5-7B-Instruct")
- `downloads` — download count
- `likes` — community approval
- `tags` — capability tags
- `pipeline_tag` — task type (text-generation, etc.)

## 2. Dataset Search

```bash
curl -s --max-time 20 "https://huggingface.co/api/datasets?search=KEYWORD&limit=10"
```

Example:
```bash
curl -s --max-time 20 "https://huggingface.co/api/datasets?search=AdvBench&limit=10"
```

Response items: `id`, `downloads`, `likes`, `tags`.

## 3. Exact Model Info

```bash
curl -s --max-time 20 "https://huggingface.co/api/models/MODEL_ID"
```

Example:
```bash
curl -s --max-time 20 "https://huggingface.co/api/models/Qwen/Qwen2.5-7B-Instruct"
```

Returns detailed info:
- `id` — full model ID
- `downloads` — total downloads
- `likes` — likes count
- `config` — model configuration (architecture, hidden size, etc.)
- `siblings` — list of files in the repo (check for safetensors, config.json)
- `gated` — whether access requires approval

## 4. Paper Search (HF Papers)

```bash
curl -s --max-time 20 "https://huggingface.co/api/papers?search=KEYWORD"
```

Example:
```bash
curl -s --max-time 20 "https://huggingface.co/api/papers?search=jailbreak+LLM"
```

Useful for finding papers that have associated model/code repos on HuggingFace.

## 5. Download Commands

### CLI (preferred):
```bash
huggingface-cli download Qwen/Qwen2.5-7B-Instruct --local-dir ./assets/models/Qwen2.5-7B-Instruct
```

For datasets:
```bash
huggingface-cli download --repo-type dataset DATASET_ID --local-dir ./assets/data/name
```

### Python:
```python
from huggingface_hub import snapshot_download
snapshot_download("Qwen/Qwen2.5-7B-Instruct", local_dir="./assets/models/Qwen2.5-7B-Instruct")
```

### Selective download (large models):
```bash
huggingface-cli download Qwen/Qwen2.5-7B-Instruct --include "*.safetensors" --local-dir ./assets/models/Qwen2.5-7B-Instruct
```

## 6. Output Format

Append to `docs/assets.md`:

```markdown
| Asset | Type | Platform | ID | Downloads | Download Command | Status |
|-------|------|----------|----|-----------|-----------------|--------|
| Qwen2.5-7B-Instruct | model | HuggingFace | Qwen/Qwen2.5-7B-Instruct | 2.1M | `huggingface-cli download Qwen/Qwen2.5-7B-Instruct --local-dir ./assets/models/Qwen2.5-7B-Instruct` | pending |
```

## 7. Gated Models

If `gated` is true in the model info:
- Note in assets.md status: `gated — requires HF login + access request`
- Check if the same model is available ungated on ModelScope (`auto-research-s1-modelscope-query`)
- Prefer ungated alternatives when available

## 8. Search Tips

- Use specific model names over generic keywords (e.g., "Qwen2.5-7B" not "chinese llm")
- Add `&filter=text-generation` to narrow by task
- Sort by downloads to find the canonical version among duplicates
- Check `pipeline_tag` matches expected use (text-generation for LLMs)
