---
name: auto-research-s2-asset-download
description: Guide for downloading models, datasets, and baseline repos for Stage 2 experiments
metadata:
  version: "0.1"
---

# Asset Download Guide

## Model Download

### HuggingFace (CLI)

```bash
huggingface-cli download MODEL_ID --local-dir ./models/NAME
# Example:
huggingface-cli download Qwen/Qwen3-4B --local-dir ./models/qwen3-4b
```

### ModelScope (CLI)

```bash
modelscope download --model 'NS/NAME' --local_dir './models/NAME'
# Example:
modelscope download --model 'Qwen/Qwen3-4B' --local_dir './models/qwen3-4b'
```

### Python API

```python
from huggingface_hub import snapshot_download
snapshot_download(repo_id="Qwen/Qwen3-4B", local_dir="./models/qwen3-4b")
```

### Proxy (if direct download fails)

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
huggingface-cli download MODEL_ID --local-dir ./models/NAME
```

## Dataset Download

### HuggingFace (CLI)

```bash
huggingface-cli download --repo-type dataset DS_ID --local-dir ./data/NAME
# Example:
huggingface-cli download --repo-type dataset Anthropic/hh-rlhf --local-dir ./data/hh-rlhf
```

### ModelScope (CLI)

```bash
modelscope download --dataset 'NS/NAME' --local_dir './data/NAME'
```

### Python (HuggingFace datasets)

```python
from datasets import load_dataset
ds = load_dataset("DS_ID", split="train")
ds.save_to_disk("./data/NAME")
```

### Large datasets — selective download

```python
from huggingface_hub import hf_hub_download
# Download only specific files
hf_hub_download(repo_id="DS_ID", filename="train.jsonl",
                repo_type="dataset", local_dir="./data/NAME")
```

## Baseline Repo Clone

```bash
git clone https://github.com/USER/REPO.git baselines/NAME

# With proxy if needed:
http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 \
  git clone https://github.com/USER/REPO.git baselines/NAME
```

After cloning:
```bash
cd baselines/NAME
pip install -r requirements.txt  # or: pip install -e .
```

## Verification

After each download, verify the asset is usable:

### Model verification

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
tok = AutoTokenizer.from_pretrained("./models/NAME", trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained("./models/NAME", trust_remote_code=True, device_map="auto")
print(model.generate(**tok("Hello", return_tensors="pt").to(model.device), max_new_tokens=10))
```

### Dataset verification

```python
from datasets import load_from_disk, load_dataset
ds = load_from_disk("./data/NAME")  # or load_dataset("json", data_files="./data/NAME/*.jsonl")
print(f"Splits: {list(ds.keys()) if hasattr(ds, 'keys') else 'single'}")
print(f"Size: {len(ds)} samples")
print(ds[0])  # inspect first sample
```

### Baseline verification

```bash
cd baselines/NAME
python main.py --help  # or: python -m NAME --help
# Check that it runs without import errors
```

## Output

After all assets verified, update `docs/stage2_progress.md`:

```markdown
## Asset Checklist
- [x] Model: Qwen3-4B → models/qwen3-4b (verified: loads + generates)
- [x] Dataset: AdvBench → data/advbench (verified: 520 samples, 1 split)
- [x] Baseline: GCG → baselines/gcg (verified: --help works)
- [ ] Model: Llama-3-8B → downloading...
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Connection timeout | Set proxy env vars, retry |
| Disk full | Check with `df -h`, use `--include` to download subset |
| Model won't load | Check `trust_remote_code=True`, verify GPU memory |
| Dataset schema mismatch | Check version/tag, compare with paper's stated version |
