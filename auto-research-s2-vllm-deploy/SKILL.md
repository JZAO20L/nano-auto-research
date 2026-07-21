---
name: auto-research-s2-vllm-deploy
description: Deploy LLMs and VLMs as OpenAI-compatible API servers using vLLM. Covers basic deployment, LoRA adapter serving, text and multimodal inference calls, and performance tuning. Use when setting up model inference servers for experiments, evaluation, or reward computation.
---

# vLLM Model Deployment

## Version Requirements

| Package | Version | Notes |
|---|---|---|
| vLLM | >= 0.17.0, <= 0.25.1 | TRL integration requires this range |
| Python | >= 3.10 | 3.12 recommended |
| PyTorch | >= 2.4.0 | Auto-installed with vLLM |
| CUDA | >= 12.1 | For NVIDIA GPUs |
| transformers | >= 4.50.0 | For latest model support |

### Installation

```bash
# Recommended: uv for fast install
uv venv
source .venv/bin/activate
uv pip install -U vllm --torch-backend=auto

# Or pip
pip install -U vllm

# Verify
python -c "import vllm; print(vllm.__version__)"
```

---

## Basic Deployment

### Text Model (e.g., Qwen3-4B)

```bash
vllm serve Qwen/Qwen3-4B \
  --port 8000 \
  --tensor-parallel-size 1 \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.9
```

### Multi-GPU (Tensor Parallelism)

```bash
vllm serve Qwen/Qwen3-14B \
  --port 8000 \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.9
```

### Specify GPUs

```bash
CUDA_VISIBLE_DEVICES=2,3 vllm serve Qwen/Qwen3-4B --port 8001
```

### Key Server Arguments

| Argument | Default | Description |
|---|---|---|
| `--port` | 8000 | Server port |
| `--tensor-parallel-size` | 1 | Shard model across N GPUs |
| `--max-model-len` | model max | Limit context length (saves KV cache) |
| `--gpu-memory-utilization` | 0.9 | GPU memory fraction for model + KV cache |
| `--enable-prefix-caching` | off | Reuse KV for shared prefixes |
| `--dtype` | auto | Model precision (auto, float16, bfloat16) |
| `--enforce-eager` | off | Disable CUDA graph (for debugging) |
| `--reasoning-parser` | none | Parse thinking tokens (use `qwen3` for Qwen3) |
| `--default-chat-template-kwargs` | none | e.g., `'{"enable_thinking": false}'` |

### Disable Thinking Mode (Qwen3)

```bash
vllm serve Qwen/Qwen3-4B \
  --reasoning-parser qwen3 \
  --default-chat-template-kwargs '{"enable_thinking": false}'
```

---

## LoRA Adapter Deployment

### Serve Base Model + LoRA

```bash
vllm serve Qwen/Qwen3-4B \
  --port 8000 \
  --enable-lora \
  --lora-modules my-lora=/path/to/lora/adapter \
  --max-lora-rank 64 \
  --max-model-len 4096
```

### Multiple LoRA Adapters

```bash
vllm serve Qwen/Qwen3-4B \
  --port 8000 \
  --enable-lora \
  --lora-modules \
    lora-a=/path/to/lora_a \
    lora-b=/path/to/lora_b \
    lora-c=/path/to/lora_c \
  --max-lora-rank 64 \
  --max-loras 3
```

### Call with LoRA (specify model name = LoRA name)

```python
from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

# Call base model
response = client.chat.completions.create(
    model="Qwen/Qwen3-4B",  # base model name
    messages=[{"role": "user", "content": "Hello"}],
)

# Call with LoRA adapter
response = client.chat.completions.create(
    model="my-lora",  # LoRA module name
    messages=[{"role": "user", "content": "Hello"}],
)
```

### LoRA Arguments

| Argument | Default | Description |
|---|---|---|
| `--enable-lora` | off | Enable LoRA support |
| `--lora-modules` | none | `name=path` pairs |
| `--max-lora-rank` | 16 | Max LoRA rank supported |
| `--max-loras` | 1 | Max concurrent LoRA adapters |

---

## Text Inference Call

### OpenAI SDK

```python
from openai import OpenAI

client = OpenAI(
    api_key="EMPTY",
    base_url="http://localhost:8000/v1",
    timeout=300,
)

# Chat completion
response = client.chat.completions.create(
    model="Qwen/Qwen3-4B",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing in one sentence."},
    ],
    max_tokens=512,
    temperature=0.7,
)
print(response.choices[0].message.content)

# Batch concurrent calls
import asyncio
from openai import AsyncOpenAI

async_client = AsyncOpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

async def batch_call(prompts: list[str]) -> list[str]:
    tasks = [
        async_client.chat.completions.create(
            model="Qwen/Qwen3-4B",
            messages=[{"role": "user", "content": p}],
            max_tokens=512,
        )
        for p in prompts
    ]
    responses = await asyncio.gather(*tasks)
    return [r.choices[0].message.content for r in responses]

results = asyncio.run(batch_call(["prompt1", "prompt2", "prompt3"]))
```

### curl

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-4B",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 256
  }'
```

---

## Multimodal Inference Call

### Image Input

```python
from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-VL-3B-Instruct",
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image_url",
                "image_url": {"url": "https://example.com/image.png"}
            },
            {
                "type": "text",
                "text": "Describe this image."
            }
        ]
    }],
    max_tokens=1024,
)
print(response.choices[0].message.content)
```

### Local Image (base64)

```python
import base64

with open("local_image.png", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

messages = [{
    "role": "user",
    "content": [
        {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}},
        {"type": "text", "text": "What is in this image?"}
    ]
}]
```

### Deploy VLM

```bash
vllm serve Qwen/Qwen2.5-VL-3B-Instruct \
  --port 8000 \
  --max-model-len 8192 \
  --enforce-eager \
  --vllm-model-impl transformers
```

For large VLMs with data-parallel vision encoding:
```bash
vllm serve Qwen/Qwen3.5-397B-A17B-FP8 \
  -dp 8 \
  --enable-expert-parallel \
  --mm-encoder-tp-mode data \
  --mm-processor-cache-type shm \
  --reasoning-parser qwen3 \
  --enable-prefix-caching
```

---

## Performance Tuning

### Throughput-Focused (high concurrency)

```bash
vllm serve Qwen/Qwen3-4B \
  --enable-prefix-caching \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.95
```

### Latency-Focused (low concurrency)

```bash
vllm serve Qwen/Qwen3-4B \
  --speculative-config '{"method": "mtp", "num_speculative_tokens": 2}' \
  --max-model-len 4096
```

### Text-Only Mode (skip vision encoder for VLMs)

```bash
vllm serve Qwen/Qwen3.5-397B-A17B-FP8 \
  --language-model-only \
  --enable-expert-parallel
```

### Benchmarking

```bash
vllm bench serve \
  --backend openai-chat \
  --endpoint /v1/chat/completions \
  --model Qwen/Qwen3-4B \
  --dataset-name random \
  --random-input-len 512 \
  --random-output-len 256 \
  --num-prompts 200 \
  --request-rate 10
```

---

## Multi-Server Setup (Typical Experiment)

For RL training experiments requiring multiple models:

```bash
# GPU 0: Policy model (for vLLM generation in TRL training)
CUDA_VISIBLE_DEVICES=0 trl vllm-serve --model Qwen/Qwen3-4B --port 8000

# GPU 1: Target model (attack target)
CUDA_VISIBLE_DEVICES=1 vllm serve Qwen/Qwen3-4B --port 8001 --max-model-len 8192

# GPU 1: Guard model (reward/safety judge, shares GPU with target)
CUDA_VISIBLE_DEVICES=1 vllm serve Qwen/Qwen3Guard-Gen-4B --port 8002 --max-model-len 2048 --gpu-memory-utilization 0.4
```

Note: When sharing a GPU, adjust `--gpu-memory-utilization` so total < 1.0.

---

## Common Issues

1. **Port conflict**: Check with `lsof -i :8000`, kill existing process or use different port
2. **OOM on startup**: Lower `--gpu-memory-utilization` (0.8) or reduce `--max-model-len`
3. **CUDA graph error (Mamba models)**: Add `--max-cudagraph-capture-size 256`
4. **LoRA not loading**: Ensure `--max-lora-rank` >= your adapter's rank
5. **Slow first request**: CUDA graph compilation is normal; subsequent requests are fast
6. **Model not found**: Check HF cache or pass local path: `vllm serve /path/to/model`
