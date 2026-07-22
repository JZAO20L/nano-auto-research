---
name: auto-research-s2-model-call
description: Guide for calling LLMs via local vLLM server or remote API with OpenAI-compatible interface
metadata:
  version: "0.1"
---

# Model Call Guide

## Configuration

All model endpoints are defined in `project_config.yaml` at project root (under the `models:` key).

### Config: project_config.yaml

The model configuration is defined in `project_config.yaml` at the project root. Each model entry has:

**Local models** (`type: local`):
```yaml
- name: Qwen3-4B
  type: local
  path: /data/models/Qwen3-4B
  description: "Backbone for training"
```
- Call via vLLM: first deploy with `vllm serve {path}`, then use the resulting endpoint
- Reference `auto-research-s2-vllm-deploy` for deployment details

**API models** (`type: api`):
```yaml
- name: qwen-max
  type: api
  url: https://dashscope.aliyuncs.com/compatible-mode/v1
  api_key: sk-xxx
  max_concurrency: 10
  rpm: 60
  description: "Remote baseline"
```
- Call directly via OpenAI-compatible client with the provided `url` and `api_key`
- `max_concurrency`: maximum number of concurrent requests to this endpoint
- `rpm`: requests per minute limit for this endpoint

### Connectivity Test

Before any experiment, test each API model:
```python
from openai import OpenAI
client = OpenAI(base_url="{url}", api_key="{api_key}")
resp = client.chat.completions.create(
    model="{name}",
    messages=[{"role": "user", "content": "Hello"}],
    max_tokens=5
)
print(resp.choices[0].message.content)  # Should print a short response
```
If this fails, report the error to the user. Do NOT silently skip the model.

### If config missing

Stop and prompt the user:
> `project_config.yaml` not found. Please create it with your model endpoints. See template above.

Do NOT hardcode keys or URLs in code.

### Loading config

```python
import yaml
from pathlib import Path

def load_model_config(name: str) -> dict:
    cfg_path = Path("project_config.yaml")
    assert cfg_path.exists(), "project_config.yaml not found — ask user to create it"
    cfg = yaml.safe_load(cfg_path.read_text())
    for model in cfg["models"]:
        if model["name"] == name:
            return model
    raise ValueError(f"Model '{name}' not found in project_config.yaml")
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

## Concurrency & Rate Limiting

For API models, always read `max_concurrency` and `rpm` from the model's config entry in `project_config.yaml`:

- Use `asyncio.Semaphore(max_concurrency)` to limit concurrent requests
- Use a rate limiter (e.g., `time.sleep(60/rpm)` between requests or a token bucket) to respect RPM

```python
import asyncio

async def batch_call_with_limits(messages_list, model_cfg):
    sem = asyncio.Semaphore(model_cfg.get("max_concurrency", 5))
    rpm = model_cfg.get("rpm", 60)
    interval = 60.0 / rpm

    async def call_one(msgs):
        async with sem:
            await asyncio.sleep(interval)
            return await async_call(msgs, model_cfg)

    return await asyncio.gather(*[call_one(m) for m in messages_list])
```

### General Concurrency Guidelines

| Endpoint | Safe concurrency | Notes |
|----------|-----------------|-------|
| Local vLLM (single GPU) | 64–128 | vLLM handles batching internally |
| Local vLLM (multi-GPU) | 128–256 | Scale with tensor parallelism |
| DashScope remote | per `max_concurrency` in config | Respect `rpm` setting |
| OpenAI API | per `max_concurrency` in config | Respect `rpm` setting |

For local vLLM endpoints (no `max_concurrency`/`rpm` in config), start at 16 and increase until errors appear, then back off.

## Starting Local vLLM Server

```bash
# GPU 0: policy model
CUDA_VISIBLE_DEVICES=0 vllm serve ./models/qwen3-4b --port 8000 --tensor-parallel-size 1 &

# GPU 1: target model
CUDA_VISIBLE_DEVICES=1 vllm serve ./models/qwen3-4b --port 8001 --tensor-parallel-size 1 &
```

(legacy: `python -m vllm.entrypoints.openai.api_server`)

Wait for server ready: `curl http://localhost:8000/v1/models`
