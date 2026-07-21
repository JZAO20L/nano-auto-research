---
name: auto-research-s2-model-training
description: Guide for training LLMs with TRL (GRPO, SFT, DPO) including custom reward functions, vLLM-accelerated generation, LoRA/PEFT, and multi-GPU distributed training. Use when setting up RL training experiments, designing reward functions, or configuring training infrastructure.
---

# Model Training with TRL

## Framework Choice

Use **TRL** (HuggingFace) for all training that involves custom reward logic. TRL's reward function is a plain Python callable — ideal for research-grade reward design (adaptive weighting, multi-dimensional judge, variance-based scheduling).

Reference docs: `docs/docs/trl/` (grpo_trainer.md, vllm_integration.md, distributing_training.md, peft_integration.md)

## Version Requirements

| Package | Version | Notes |
|---|---|---|
| trl | >= 0.19.0 | GRPOTrainer, DistillationTrainer |
| transformers | >= 4.50.0 | SwanLab native integration, latest model support |
| accelerate | >= 1.0.0 | Multi-GPU distributed training |
| peft | >= 0.13.0 | LoRA/QLoRA support |
| vllm | >= 0.17.0, <= 0.25.1 | TRL vLLM integration (server/colocate mode) |
| datasets | >= 3.0.0 | Dataset loading |
| torch | >= 2.4.0 | PyTorch backend |
| bitsandbytes | >= 0.43.0 | QLoRA (4-bit quantization), optional |
| swanlab | latest | Experiment tracking, optional |
| deepspeed | >= 0.14.0 | ZeRO Stage 3 for large models (>14B), optional |

### Installation

```bash
# Core
uv pip install trl[vllm,peft] accelerate datasets

# Optional
uv pip install bitsandbytes   # QLoRA
uv pip install swanlab        # experiment tracking
uv pip install deepspeed      # large model training
```

---

## GRPO Training (Core)

### Minimal Setup

```python
from datasets import load_dataset
from trl import GRPOTrainer, GRPOConfig

dataset = load_dataset("path/to/dataset", split="train")

training_args = GRPOConfig(
    output_dir="./output",
    per_device_train_batch_size=4,
    num_generations=4,          # completions per prompt
    max_completion_length=512,
    learning_rate=1e-6,
    num_train_epochs=1,
)

trainer = GRPOTrainer(
    model="Qwen/Qwen3-4B",
    args=training_args,
    reward_funcs=my_reward_fn,
    train_dataset=dataset,
)
trainer.train()
```

Launch: `accelerate launch train.py`

### Custom Reward Function

Reward function signature — accepts keyword args, returns list of floats:

```python
def my_reward_fn(prompts, completions, completion_ids, **kwargs):
    """
    Args received from GRPOTrainer:
        prompts: list[str] — input prompts
        completions: list[str] — generated completions
        completion_ids: list[list[int]] — tokenized completions
        trainer_state: TrainerState — current training step, epoch, etc.
        **kwargs: any extra dataset columns (e.g., ground_truth, task)
    Returns:
        list[float] — one reward per completion
    """
    rewards = []
    for prompt, completion in zip(prompts, completions):
        score = compute_score(prompt, completion)
        rewards.append(score)
    return rewards
```

Key points:
- Return `None` for a sample to skip it (multi-task scenarios)
- Multiple reward functions: pass as list, combine with `reward_weights` in GRPOConfig
- Async reward functions (`async def`) are supported — multiple async funcs run concurrently
- `trainer_state` enables dynamic rewards (curriculum, adaptive weighting based on step)
- `log_metric(name, value)` inside reward fn to log custom metrics alongside training

### Adaptive/Dynamic Reward Pattern

For rewards that change based on training progress (e.g., adaptive weight between ASR and judge):

```python
def adaptive_reward_fn(prompts, completions, trainer_state, **kwargs):
    step = trainer_state.global_step
    # Compute component rewards
    asr_rewards = compute_asr(completions)
    judge_rewards = compute_judge(completions)
    # Dynamic weight based on variance / EMA / schedule
    weight = compute_adaptive_weight(asr_rewards, judge_rewards, step)
    return [w * a + (1 - w) * j for w, a, j in zip(weight, asr_rewards, judge_rewards)]
```

### External Model as Reward (vLLM server)

When reward requires calling another model (judge, target, guard):

```python
import httpx

JUDGE_URL = "http://localhost:8001/v1/chat/completions"

async def judge_reward_fn(prompts, completions, **kwargs):
    """Async reward — calls external judge model via HTTP."""
    async with httpx.AsyncClient() as client:
        tasks = []
        for prompt, completion in zip(prompts, completions):
            tasks.append(client.post(JUDGE_URL, json={
                "model": "judge-model",
                "messages": [{"role": "user", "content": build_judge_prompt(prompt, completion)}],
                "max_tokens": 256,
            }))
        responses = await asyncio.gather(*tasks)
    return [parse_score(r.json()) for r in responses]
```

---

## vLLM-Accelerated Generation

Generation is the bottleneck in online RL. Use vLLM to speed it up.

### Server Mode (recommended for multi-GPU)

vLLM on dedicated GPUs, training on separate GPUs:

```bash
# Terminal 1: Start vLLM server on GPU 0-1
CUDA_VISIBLE_DEVICES=0,1 trl vllm-serve --model Qwen/Qwen3-4B --tensor-parallel-size 2 --port 8000

# Terminal 2: Train on GPU 2-3
CUDA_VISIBLE_DEVICES=2,3 accelerate launch train.py
```

Training config:
```python
training_args = GRPOConfig(
    use_vllm=True,
    vllm_mode="server",
    # vllm_server_host="localhost",  # default
    # vllm_server_port=8000,         # default
)
```

### Colocate Mode (single GPU or memory-constrained)

vLLM runs inside trainer process, shares GPU:
```python
training_args = GRPOConfig(
    use_vllm=True,  # vllm_mode="colocate" by default
)
```

### Important Notes

- Server and trainer MUST use different GPUs (NCCL conflict otherwise)
- `vllm_gpu_memory_utilization` (default 0.9) — lower if OOM
- `max_model_len` — set explicitly to avoid KV cache waste (e.g., 2048 for short completions)
- TRL handles weight sync: after each training step, updated weights are pushed to vLLM server
- Truncated Importance Sampling is enabled by default to correct train-inference mismatch

---

## LoRA / PEFT

### Setup

```python
from peft import LoraConfig
from trl import GRPOTrainer, GRPOConfig

peft_config = LoraConfig(
    r=32,
    lora_alpha=16,
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
)

trainer = GRPOTrainer(
    model="Qwen/Qwen3-4B",
    args=GRPOConfig(learning_rate=2e-5),  # 10x higher LR for LoRA
    reward_funcs=my_reward_fn,
    train_dataset=dataset,
    peft_config=peft_config,
)
```

### Notes

- LoRA LR should be ~10x full fine-tuning LR (2e-5 vs 2e-6)
- For QLoRA (4-bit): `pip install bitsandbytes`, add `load_in_4bit=True` to model loading
- Save: `trainer.save_model("./lora_output")` — saves adapter only
- Merge: `model.merge_and_unload()` for deployment

---

## Multi-GPU Distributed Training

### Accelerate (data parallelism)

```bash
# Configure once
accelerate config

# Launch
accelerate launch --num_processes 4 train.py
```

Effective batch size = `per_device_train_batch_size × num_gpus × gradient_accumulation_steps`

### Typical GPU Assignment (4-GPU setup)

```
GPU 0-1: Policy model training (accelerate data parallel)
GPU 2:   vLLM server for generation (policy copy)
GPU 3:   Target model + Guard model (reward computation)
```

Or for training-only (no separate vLLM):
```
GPU 0-1: Policy training (colocate vLLM)
GPU 2-3: Target + Guard (external reward servers)
```

### DeepSpeed ZeRO (for large models >14B)

```bash
accelerate launch --config_file deepspeed_zero3.yaml train.py
```

---

## Key GRPOConfig Parameters

| Parameter | Default | Notes |
|---|---|---|
| `num_generations` | 8 | Completions per prompt (group size G) |
| `max_completion_length` | 256 | Max tokens per completion |
| `per_device_train_batch_size` | 8 | Prompts per GPU per step |
| `learning_rate` | 1e-6 | Lower for full FT, higher for LoRA |
| `beta` | 0.0 | KL penalty (0 = no KL, common practice) |
| `scale_rewards` | True | Normalize by std (False removes difficulty bias) |
| `loss_type` | "grpo" | Options: "grpo", "dapo", "dr_grpo", "sapo" |
| `num_iterations` | 1 | Policy updates per generation (μ) |
| `use_vllm` | False | Enable vLLM generation |
| `vllm_mode` | "colocate" | "colocate" or "server" |
| `reward_weights` | None | Weights for multiple reward functions |

---

## SFT Training (for baseline / warmup)

```python
from trl import SFTTrainer, SFTConfig

trainer = SFTTrainer(
    model="Qwen/Qwen3-4B",
    args=SFTConfig(
        output_dir="./sft_output",
        per_device_train_batch_size=4,
        learning_rate=2e-5,
        num_train_epochs=3,
        max_seq_length=2048,
    ),
    train_dataset=dataset,
    peft_config=peft_config,  # optional LoRA
)
trainer.train()
```

---

## On-Policy Distillation (DistillationTrainer)

For compressing a teacher model into a smaller student. Experimental in TRL.

```python
from datasets import load_dataset
from trl.experimental.distillation import DistillationConfig, DistillationTrainer

dataset = load_dataset("openai/gsm8k", "main", split="train")
dataset = dataset.map(
    lambda x: {"messages": [{"role": "user", "content": x["question"]}]},
    remove_columns=dataset.column_names,
)

config = DistillationConfig(
    output_dir="results/distill",
    num_train_epochs=1,
    bf16=True,
    lmbda=1.0,    # fully on-policy (student generates)
    beta=1.0,     # reverse KL
)

trainer = DistillationTrainer(
    model="Qwen/Qwen2.5-1.5B-Instruct",          # student
    teacher_model="Qwen/Qwen2.5-7B-Instruct",     # teacher
    args=config,
    train_dataset=dataset,
)
trainer.train()
```

Key parameters:
- `lmbda`: 0.0 = off-policy (dataset completions), 1.0 = on-policy (student generates). On-policy generally better.
- `beta`: 0.0 = forward KL, 1.0 = reverse KL. Interpolates between.
- `loss_top_k`: top tokens for KL loss. 0 = full vocab (local teacher only).

### External Teacher Server (for large teachers)

```python
config = DistillationConfig(
    output_dir="distilled-model",
    use_teacher_server=True,
    teacher_model_server_url="http://teacher-host:8000",
    loss_top_k=1,       # required with teacher server when beta > 0
    beta=1.0,
    lmbda=1.0,
)

trainer = DistillationTrainer(
    model="Qwen/Qwen3-4B",
    args=config,
    train_dataset=dataset,
)
```

---

## GSPO-token (Experimental)

Token-level variant of Group Sequence Policy Optimization. Use when you have per-token advantages.

```python
from trl.experimental.gspo_token import GRPOTrainer
from trl import GRPOConfig

training_args = GRPOConfig(
    importance_sampling_level="sequence_token",
    # ... other args
)
```

Note: Only differs from standard GSPO when per-token advantage `A_{i,t}` varies with `t`. If advantage is constant across tokens (standard GRPO), GSPO-token is equivalent to GSPO.

---

## Reward Model Training (RewardTrainer)

For training a reward model from preference pairs (chosen vs rejected).

```python
from trl import RewardTrainer, RewardConfig
from datasets import load_dataset

dataset = load_dataset("trl-lib/ultrafeedback_binarized", split="train")

trainer = RewardTrainer(
    model="Qwen/Qwen3-0.6B",
    args=RewardConfig(
        output_dir="./reward_model",
        per_device_train_batch_size=4,
        learning_rate=1e-5,
        center_rewards_coefficient=1e-2,  # center rewards around zero
    ),
    train_dataset=dataset,
)
trainer.train()
```

### Dataset Format

```python
# Conversational preference (explicit prompt)
{"prompt": [{"role": "user", "content": "What color is the sky?"}],
 "chosen": [{"role": "assistant", "content": "It is blue."}],
 "rejected": [{"role": "assistant", "content": "It is green."}]}

# Standard preference (implicit prompt)
{"chosen": "The sky is blue.",
 "rejected": "The sky is green."}
```

### With LoRA

```python
from peft import LoraConfig

trainer = RewardTrainer(
    model="Qwen/Qwen3-4B",
    train_dataset=dataset,
    peft_config=LoraConfig(modules_to_save=["score"]),  # include score head
)
```

Key metrics logged: `accuracy` (chosen > rejected), `margin` (reward gap), `mean_reward`.

---

## SwanLab Integration (Experiment Tracking)

SwanLab provides experiment tracking and visualization for TRL training.

### One-line Integration (transformers >= 4.50.0)

```python
from trl import GRPOConfig

training_args = GRPOConfig(
    ...,
    report_to="swanlab",  # that's it
)
```

Custom project name:
```python
import os
os.environ["SWANLAB_PROJECT"] = "my-grpo-experiment"
```

### SwanLabCallback (transformers < 4.50.0 or more control)

```python
from swanlab.integration.transformers import SwanLabCallback
from trl import GRPOTrainer, GRPOConfig

swanlab_callback = SwanLabCallback(
    project="jailbreak-rl",
    experiment_name="ahr-grpo-exp1",
    description="Adaptive hybrid reward GRPO training",
)

trainer = GRPOTrainer(
    ...,
    args=GRPOConfig(report_to="none"),
    callbacks=[swanlab_callback],
)
```

### Setup

```bash
pip install swanlab
swanlab login  # paste API key from https://swanlab.cn/settings
```

API Key: `lIrr4ZEyrTCWsWGQE9szs`

---

## vLLM Integration Details

### Server Startup Options

```bash
trl vllm-serve --model Qwen/Qwen3-4B \
    --tensor-parallel-size 2 \
    --port 8000 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 4096 \
    --enable-prefix-caching \
    --dtype auto
```

Key server args:
- `--tensor-parallel-size`: shard model across N GPUs
- `--gpu-memory-utilization`: KV cache budget (0.9 default, lower if OOM)
- `--max-model-len`: limit context length to save KV cache
- `--enable-prefix-caching`: reuse KV for shared prefixes (helpful for same-system-prompt batches)
- `--enforce-eager`: disable CUDA graph (debug, slower)

### Weight Sync Mechanism

When using vLLM with TRL:
1. Trainer generates completions via vLLM server
2. Trainer computes loss and updates weights
3. Trainer pushes updated weights to vLLM server (`update_named_param`)
4. Next generation step uses updated policy

This keeps vLLM's policy in sync with training without restarting the server.

### Importance Sampling Correction

vLLM and training engines produce slightly different outputs (precision, hardware optimizations). TRL corrects this with Truncated Importance Sampling (TIS) by default:
- Ratios > `vllm_importance_sampling_cap` are clipped
- Disable with `vllm_importance_sampling_correction=False` (not recommended)
- Alternative: Masked Importance Sampling (MIS) via `vllm_importance_sampling_mode`

### VLM Support (Vision-Language Models)

```bash
CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES=0 trl vllm-serve \
    --model Qwen/Qwen2.5-VL-3B-Instruct \
    --tensor-parallel-size 1 \
    --port 8000 \
    --enforce_eager \
    --vllm_model_impl transformers
```

---

## Agent Training with GRPO (Tools + Environments)

GRPO supports training agents that call tools during generation and learn from outcomes. Directly relevant to agentic RL research.

### Tools (Stateless)

Each tool is a Python function with type hints + Google-style docstring:

```python
from trl import GRPOTrainer, GRPOConfig

def query_database(sql_command: str) -> list[tuple]:
    """
    Execute a read-only SQL command on the database.

    Args:
        sql_command: The SQL command to execute.

    Returns:
        A list of tuples containing the query results.
    """
    conn = sqlite3.connect("file:data.db?mode=ro", uri=True)
    cursor = conn.cursor()
    cursor.execute(sql_command)
    results = cursor.fetchall()
    conn.close()
    return results

trainer = GRPOTrainer(
    model="Qwen/Qwen3-1.7B",
    tools=[query_database],  # list of tool functions
    reward_funcs=[correctness_reward, structure_reward],
    args=GRPOConfig(
        max_steps=400,
        max_completion_length=1024,
        use_vllm=True,
        vllm_mode="colocate",
        chat_template_kwargs={"enable_thinking": False},
    ),
    train_dataset=dataset,
)
trainer.train()
```

Key points:
- Tools can be sync or async (async tools run concurrently)
- TRL auto-patches chat template for Qwen3/DeepSeek-V3 when tools are enabled
- Multiple reward functions can evaluate different aspects (correctness, structure, query quality)

### Environments (Stateful)

For per-rollout state, use `environment_factory` (requires `transformers>=5.2.0`):

```python
import random
from trl import GRPOConfig, GRPOTrainer

class MyEnv:
    def reset(self, **kwargs) -> str | None:
        """Called at start of each rollout. Return string = prompt."""
        self.counter = 0
        self.target = random.randint(1, 6)
        return f"Increment the counter by {self.target}."

    def get_reward(self) -> float:
        """Optional: environment scores itself from internal state."""
        return float(self.counter == self.target)

    # Public methods below are exposed as tools
    def increment(self, step: int) -> int:
        """
        Increment the internal counter.

        Args:
            step: Value to add to the counter.

        Returns:
            The updated counter value.
        """
        self.counter += step
        return self.counter

trainer = GRPOTrainer(
    model="Qwen/Qwen3-0.6B",
    args=GRPOConfig(max_steps=1000, chat_template_kwargs={"enable_thinking": False}),
    environment_factory=MyEnv,
)
trainer.train()
```

Environment rules:
- `reset()` (required): called per rollout, returns prompt string or None
- `get_reward()` (optional): environment-owned reward, summed with `reward_funcs`
- All other public methods → exposed as tools to the model
- Can combine with external `train_dataset` (row columns passed to `reset()` as kwargs)
- Can combine `tools=[...]` alongside `environment_factory`

### Agent Reward Design Pattern

```python
def correctness_reward(completions, answer, **kwargs):
    """Reward correct final answers."""
    rewards = []
    for completion, ans in zip(completions, answer):
        # completion is a list of message dicts (multi-turn)
        last_content = completion[-1]["content"].lower()
        reward = 1.0 if ans.lower() in last_content else -1.0
        rewards.append(reward)
    return rewards

def tool_usage_reward(completions, **kwargs):
    """Reward efficient tool usage patterns."""
    rewards = []
    for completion in completions:
        tool_calls = sum(1 for turn in completion if turn.get("tool_calls"))
        # Penalize too many calls, reward targeted queries
        reward = -0.5 if tool_calls > 3 else 0.2 * tool_calls
        rewards.append(reward)
    return rewards
```

---

## Entropy Regularization

Prevents policy collapse by adding entropy bonus to the loss.

### Static Entropy

```python
training_args = GRPOConfig(entropy_coef=0.05, ...)
```

### Adaptive Entropy (Skywork-OR1 style)

Coefficient auto-adjusts based on target entropy:

```python
training_args = GRPOConfig(
    entropy_coef=0.01,          # initial coefficient
    use_adaptive_entropy=True,
    entropy_target=5.0,         # target mean per-token entropy (nats)
    entropy_coef_delta=0.005,   # step size per optimizer step
    entropy_coef_min=0.0,
    entropy_coef_max=1.0,
    ...
)
```

Typical per-token entropy: 2–10 nats. Set `entropy_target` close to early-training entropy (logged as `entropy` metric).

---

## Accelerate Config Templates

Reference configs for distributed training (from `trl/examples/accelerate_configs/`):

### Multi-GPU (Data Parallel)

```yaml
# multi_gpu.yaml
compute_environment: LOCAL_MACHINE
distributed_type: MULTI_GPU
mixed_precision: 'bf16'
num_machines: 1
num_processes: 8
```

### DeepSpeed ZeRO Stage 1

```yaml
# deepspeed_zero1.yaml
compute_environment: LOCAL_MACHINE
distributed_type: DEEPSPEED
deepspeed_config:
  zero_stage: 1
  zero3_init_flag: false
  gradient_accumulation_steps: 1
mixed_precision: 'bf16'
num_processes: 8
```

### DeepSpeed ZeRO Stage 2

```yaml
# deepspeed_zero2.yaml
compute_environment: LOCAL_MACHINE
distributed_type: DEEPSPEED
deepspeed_config:
  zero_stage: 2
  zero3_init_flag: false
  offload_optimizer_device: none
  offload_param_device: none
mixed_precision: 'bf16'
num_processes: 8
```

### DeepSpeed ZeRO Stage 3 (for models >14B)

```yaml
# deepspeed_zero3.yaml
compute_environment: LOCAL_MACHINE
distributed_type: DEEPSPEED
deepspeed_config:
  zero_stage: 3
  zero3_init_flag: true
  zero3_save_16bit_model: true
  offload_optimizer_device: none
  offload_param_device: none
mixed_precision: bf16
num_processes: 8
```

### Launch Commands

```bash
# Multi-GPU
accelerate launch --config_file multi_gpu.yaml --num_processes 4 train.py

# DeepSpeed
accelerate launch --config_file deepspeed_zero3.yaml --num_processes 8 train.py

# Custom GPU count
accelerate launch --num_processes 2 train.py
```

---

## Reference Scripts & Notebooks

| Resource | Description |
|---|---|
| `trl/scripts/grpo.py` | Standard GRPO training script |
| `trl/scripts/grpo_agent.py` | GRPO agent training with tools |
| `trl/scripts/sft.py` | SFT training script |
| `trl/scripts/dpo.py` | DPO training script |
| `examples/scripts/distillation.py` | On-policy distillation |
| `examples/scripts/gspo.py` | GSPO (sequence-level importance sampling) |
| `examples/notebooks/grpo_agent.ipynb` | Agent training notebook (BioGRID QA) |
| `examples/notebooks/grpo_trl_lora_qlora.ipynb` | GRPO with QLoRA on Colab |

---

## Common Issues

1. **OOM during generation**: Reduce `num_generations` or `per_device_train_batch_size`; set `max_model_len` in vLLM
2. **Reward all zeros**: Check completion format matches reward parsing logic; log completions to debug
3. **NCCL errors**: Ensure vLLM server and trainer use different `CUDA_VISIBLE_DEVICES`
4. **Training unstable**: Try `scale_rewards=False`, reduce LR, or use `loss_type="dr_grpo"`
5. **vLLM version**: TRL requires vLLM 0.17.0–0.25.1
6. **Agent tool template**: Qwen3 auto-patched; other models may need prefix-preserving chat template
7. **Environment factory**: Requires `transformers>=5.2.0`
