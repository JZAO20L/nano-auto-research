---
name: auto-research-s2-model-call
description: Guide for calling LLMs via local vLLM server or remote API with OpenAI-compatible interface
metadata:
  version: "0.1"
---

# Model Call Guide

## Configuration

All model endpoints are defined in `model_config.yaml` at project root.

### Template

```yaml
local:
  policy:
    base_url: "http://localhost:8000/v1"
    model: "Qwen/Qwen3-4B"
    api_key: "EMPTY"
  target:
    base_url: "http://localhost:8001/v1"
    model: "Qwen/Qwen3-4B"
    api_key: "EMPTY"
remote:
  provider: "dashscope"
  base_url: "https://dashscope.aliyuncs.com/compatible-mode/v1"
  api_key: "sk-xxx"
  model: "qwen3-max"
```

### If config missing

Stop and prompt the user:
> `model_config.yaml` not found. Please create it with your model endpoints. See template above.

Do NOT hardcode keys or URLs in code.

### Loading config

```python
import yaml
from pathlib import Path

def load_model_config(role: str = "target", mode: str = "local") -> dict:
    cfg_path = Path("model_config.yaml")
    assert cfg_path.exists(), "model_config.yaml not found — ask user to create it"
    cfg = yaml.safe_load(cfg_path.read_text())
    if mode == "local":
        return cfg["local"][role]
    return cfg["remote"]
```

## Synchronous Call (OpenAI SDK)

```python
from openai import OpenAI

def call_model(prompt: str, cfg: dict, system: str = None, max_tokens: int = 512) -> str:
    client = OpenAI(api_key=cfg["api_key"], base_url=cfg["base_url"])
    messages = []
    if system:
        messages.append({"role": "system", "content": system})
    messages.append({"role": "user", "content": prompt})
    response = client.chat.completions.create(
        model=cfg["model"], messages=messages, max_tokens=max_tokens, temperature=0.7
    )
    return response.choices[0].message.content
```

## Async Batch Call

```python
from openai import AsyncOpenAI
import asyncio

async def batch_call(prompts: list[str], cfg: dict, concurrency: int = 32,
                     system: str = None, max_tokens: int = 512) -> list[str]:
    client = AsyncOpenAI(api_key=cfg["api_key"], base_url=cfg["base_url"])
    sem = asyncio.Semaphore(concurrency)

    async def call(p: str) -> str:
        async with sem:
            messages = []
            if system:
                messages.append({"role": "system", "content": system})
            messages.append({"role": "user", "content": p})
            resp = await client.chat.completions.create(
                model=cfg["model"], messages=messages, max_tokens=max_tokens
            )
            return resp.choices[0].message.content or ""

    return await asyncio.gather(*[call(p) for p in prompts])

# Usage:
# results = asyncio.run(batch_call(prompts, cfg, concurrency=64))
```

## Retry Logic

```python
import asyncio
from openai import APITimeoutError, RateLimitError

async def call_with_retry(client, messages, model, max_tokens=512, retries=3):
    for attempt in range(retries):
        try:
            resp = await client.chat.completions.create(
                model=model, messages=messages, max_tokens=max_tokens
            )
            return resp.choices[0].message.content or ""
        except (APITimeoutError, RateLimitError) as e:
            if attempt == retries - 1:
                raise
            wait = 2 ** (attempt + 1)  # 2s, 4s, 8s
            print(f"Retry {attempt+1}/{retries} after {wait}s: {e}")
            await asyncio.sleep(wait)
```

## Result Parsing

```python
import re

def parse_response(text: str) -> str:
    """Strip thinking tags and extract final answer."""
    if not text:
        return ""
    # Remove <think>...</think> blocks (Qwen3 thinking mode)
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL).strip()
    return text
```

## Concurrency Guidelines

| Endpoint | Safe concurrency | Notes |
|----------|-----------------|-------|
| Local vLLM (single GPU) | 64–128 | vLLM handles batching internally |
| Local vLLM (multi-GPU) | 128–256 | Scale with tensor parallelism |
| DashScope remote | 16–32 | Check rate limit in console |
| OpenAI API | 32–64 | Depends on tier |

When unsure, start at 16 and increase until rate-limit errors appear, then back off.

## Starting Local vLLM Server

```bash
# GPU 0: policy model
CUDA_VISIBLE_DEVICES=0 vllm serve ./models/qwen3-4b --port 8000 --tensor-parallel-size 1 &

# GPU 1: target model
CUDA_VISIBLE_DEVICES=1 vllm serve ./models/qwen3-4b --port 8001 --tensor-parallel-size 1 &
```

(legacy: `python -m vllm.entrypoints.openai.api_server`)

Wait for server ready: `curl http://localhost:8000/v1/models`
