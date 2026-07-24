---
name: auto-research-s2-embedding
description: Guide for using Qwen3-Embedding models (0.6B/4B/8B) for text embedding and retrieval tasks. Covers model loading, batch encoding, similarity computation, and integration with experiment pipelines.
metadata:
  version: "0.1"
---

# Embedding Model Usage Guide

## Model Overview

Qwen3-Embedding series provides state-of-the-art text embedding models in multiple sizes (0.6B, 4B, 8B) with:
- **Multilingual support**: 100+ languages
- **Context length**: 32K tokens
- **Embedding dimension**: Up to 4096 (user-configurable from 32-4096)
- **Instruction-aware**: Customizable instructions for task-specific performance boost

### Model Selection

| Model | Size | Embedding Dim | MTEB Score | Use Case |
|-------|------|---------------|------------|----------|
| Qwen3-Embedding-0.6B | 0.6B | 1024 | 64.33 | Fast prototyping, resource-constrained |
| Qwen3-Embedding-4B | 4B | 2560 | 69.45 | Balanced performance |
| Qwen3-Embedding-8B | 8B | 4096 | 70.58 | Maximum quality, retrieval tasks |

---

## Installation

```bash
# Requires transformers >= 4.51.0
pip install -U transformers sentence-transformers torch

# Optional: flash attention for acceleration
pip install flash-attn --no-build-isolation
```

---

## Method 1: Sentence Transformers (Recommended)

### Basic Usage

```python
from sentence_transformers import SentenceTransformer

# Load model
model = SentenceTransformer("Qwen/Qwen3-Embedding-8B")

# Optional: Enable flash attention for better performance
# model = SentenceTransformer(
#     "Qwen/Qwen3-Embedding-8B",
#     model_kwargs={"attn_implementation": "flash_attention_2", "device_map": "auto"},
#     tokenizer_kwargs={"padding_side": "left"},
# )

# Queries with instruction (recommended for retrieval tasks)
queries = [
    "What is the capital of China?",
    "Explain quantum computing",
]
query_embeddings = model.encode(queries, prompt_name="query")

# Documents (no instruction needed)
documents = [
    "The capital of China is Beijing.",
    "Quantum computing uses quantum mechanical phenomena.",
]
document_embeddings = model.encode(documents)

# Compute similarity
similarity = model.similarity(query_embeddings, document_embeddings)
print(similarity)
```

### Custom Instructions

```python
# For specific tasks, use custom instructions for 1-5% performance boost
task_instruction = "Retrieve relevant research papers about machine learning"

queries = ["attention mechanisms", "transformer architectures"]
query_embeddings = model.encode(queries, prompt=task_instruction)
```

---

## Method 2: Transformers (Low-Level Control)

```python
import torch
import torch.nn.functional as F
from transformers import AutoTokenizer, AutoModel

def last_token_pool(last_hidden_states, attention_mask):
    """Extract embedding from last token (Qwen3's pooling strategy)."""
    left_padding = (attention_mask[:, -1].sum() == attention_mask.shape[0])
    if left_padding:
        return last_hidden_states[:, -1]
    else:
        sequence_lengths = attention_mask.sum(dim=1) - 1
        batch_size = last_hidden_states.shape[0]
        return last_hidden_states[
            torch.arange(batch_size, device=last_hidden_states.device),
            sequence_lengths
        ]

def get_detailed_instruct(task_description: str, query: str) -> str:
    """Format query with task instruction."""
    return f'Instruct: {task_description}\nQuery:{query}'

# Load model
tokenizer = AutoTokenizer.from_pretrained('Qwen/Qwen3-Embedding-8B', padding_side='left')
model = AutoModel.from_pretrained('Qwen/Qwen3-Embedding-8B')

# Optional: Enable flash attention
# model = AutoModel.from_pretrained(
#     'Qwen/Qwen3-Embedding-8B',
#     attn_implementation="flash_attention_2",
#     torch_dtype=torch.float16
# ).cuda()

# Prepare inputs
task = 'Given a web search query, retrieve relevant passages that answer the query'
queries = [
    get_detailed_instruct(task, 'What is machine learning?'),
    get_detailed_instruct(task, 'Explain neural networks')
]
documents = [
    "Machine learning is a subset of AI.",
    "Neural networks are computing systems inspired by biological neural networks."
]

input_texts = queries + documents

# Tokenize
batch_dict = tokenizer(
    input_texts,
    padding=True,
    truncation=True,
    max_length=8192,  # Adjust based on your data
    return_tensors="pt",
)

# Generate embeddings
with torch.no_grad():
    outputs = model(**batch_dict)
    embeddings = last_token_pool(outputs.last_hidden_state, batch_dict['attention_mask'])

# Normalize embeddings
embeddings = F.normalize(embeddings, p=2, dim=1)

# Compute similarity scores
scores = embeddings[:2] @ embeddings[2:].T
print(scores.tolist())
```

---

## Method 3: Batch Processing (Large-Scale)

### Async Batch Encoding

```python
import asyncio
from sentence_transformers import SentenceTransformer
import torch

async def batch_encode_async(texts: list[str], model_name: str = "Qwen/Qwen3-Embedding-8B",
                             batch_size: int = 64) -> torch.Tensor:
    """Encode large text corpus asynchronously."""
    model = SentenceTransformer(model_name)

    # Process in batches to avoid OOM
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        embeddings = model.encode(batch, show_progress_bar=False)
        all_embeddings.append(embeddings)

        # Optional: yield control periodically
        if i % (batch_size * 10) == 0:
            await asyncio.sleep(0)

    return torch.cat([torch.tensor(e) for e in all_embeddings])

# Usage
texts = ["document 1", "document 2", ...] * 10000
embeddings = asyncio.run(batch_encode_async(texts, batch_size=64))
```

### Memory-Efficient Batch Processing

```python
from sentence_transformers import SentenceTransformer
import numpy as np

def encode_large_corpus(texts: list[str],
                        model_name: str = "Qwen/Qwen3-Embedding-4B",
                        batch_size: int = 32,
                        output_path: str = "embeddings.npy"):
    """Encode large corpus and save to disk."""
    model = SentenceTransformer(model_name)

    # Preallocate numpy array
    sample_emb = model.encode(["sample"])
    embedding_dim = sample_emb.shape[-1]
    embeddings = np.memmap(output_path, dtype='float32', mode='w+',
                          shape=(len(texts), embedding_dim))

    # Process in batches
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        batch_embeddings = model.encode(batch, show_progress_bar=False)
        embeddings[i:i + batch_size] = batch_embeddings

    embeddings.flush()
    return output_path
```

---

## Method 4: vLLM Deployment (Production)

### Deploy Embedding Server

```bash
# Deploy embedding model with vLLM (requires vllm >= 0.8.5)
vllm serve Qwen/Qwen3-Embedding-8B \
  --task embed \
  --port 8000 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.9
```

### Call vLLM Embedding API

```python
import requests
import torch

def get_embeddings_vllm(texts: list[str], base_url: str = "http://localhost:8000"):
    """Get embeddings from vLLM embedding server."""
    response = requests.post(
        f"{base_url}/v1/embeddings",
        json={"model": "Qwen/Qwen3-Embedding-8B", "input": texts}
    )
    return torch.tensor([item["embedding"] for item in response.json()["data"]])

# Usage
texts = ["text 1", "text 2"]
embeddings = get_embeddings_vllm(texts)
```

---

## Method 5: Text Embeddings Inference (TEI) - Docker

### Deploy with Docker

```bash
# GPU deployment
docker run --gpus all -p 8080:80 \
  -v hf_cache:/data \
  --pull always \
  ghcr.io/huggingface/text-embeddings-inference:1.7.2 \
  --model-id Qwen/Qwen3-Embedding-8B \
  --dtype float16

# CPU deployment
docker run -p 8080:80 \
  -v hf_cache:/data \
  --pull always \
  ghcr.io/huggingface/text-embeddings-inference:cpu-1.7.2 \
  --model-id Qwen/Qwen3-Embedding-8B \
  --dtype float16
```

### Call TEI API

```bash
curl http://localhost:8080/embed \
  -X POST \
  -d '{"inputs": ["Instruct: Retrieve relevant documents\nQuery: What is AI?", "Artificial intelligence mimics human cognition."]}' \
  -H "Content-Type: application/json"
```

---

## Common Use Cases in Research

### 1. Document Retrieval / RAG

```python
from sentence_transformers import SentenceTransformer
import torch.nn.functional as F

def build_retrieval_index(corpus: list[str], model_name: str = "Qwen/Qwen3-Embedding-8B"):
    """Build retrieval index from corpus."""
    model = SentenceTransformer(model_name)

    # Encode corpus
    corpus_embeddings = model.encode(corpus, show_progress_bar=True)

    def retrieve(query: str, top_k: int = 5):
        """Retrieve top-k documents for query."""
        query_embedding = model.encode([query], prompt_name="query")
        similarities = model.similarity(query_embedding, corpus_embeddings)
        top_indices = similarities[0].topk(top_k).indices.tolist()
        return [(corpus[i], similarities[0][i].item()) for i in top_indices]

    return retrieve

# Usage
corpus = ["doc1", "doc2", "doc3", ...]
retriever = build_retrieval_index(corpus)
results = retriever("machine learning papers", top_k=10)
```

### 2. Text Classification (Embedding + Classifier)

```python
from sentence_transformers import SentenceTransformer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# Prepare data
train_texts = ["text1", "text2", ...]
train_labels = [0, 1, ...]
test_texts = ["text3", "text4", ...]
test_labels = [0, 1, ...]

# Generate embeddings
model = SentenceTransformer("Qwen/Qwen3-Embedding-4B")
train_embeddings = model.encode(train_texts)
test_embeddings = model.encode(test_texts)

# Train classifier
clf = LogisticRegression(max_iter=1000)
clf.fit(train_embeddings, train_labels)

# Evaluate
predictions = clf.predict(test_embeddings)
accuracy = accuracy_score(test_labels, predictions)
print(f"Accuracy: {accuracy:.4f}")
```

### 3. Semantic Similarity Evaluation

```python
from sentence_transformers import SentenceTransformer
import torch.nn.functional as F

def compute_semantic_similarity(text1: str, text2: str,
                                model_name: str = "Qwen/Qwen3-Embedding-8B"):
    """Compute cosine similarity between two texts."""
    model = SentenceTransformer(model_name)
    embeddings = model.encode([text1, text2])
    similarity = F.cosine_similarity(
        torch.tensor(embeddings[0]).unsqueeze(0),
        torch.tensor(embeddings[1]).unsqueeze(0)
    )
    return similarity.item()

# Usage for evaluation
similarity = compute_semantic_similarity(
    "The cat sat on the mat",
    "A feline rested on the carpet"
)
print(f"Similarity: {similarity:.4f}")
```

### 4. Clustering Analysis

```python
from sentence_transformers import SentenceTransformer
from sklearn.cluster import KMeans
import numpy as np

def cluster_texts(texts: list[str], n_clusters: int = 5,
                  model_name: str = "Qwen/Qwen3-Embedding-4B"):
    """Cluster texts by semantic similarity."""
    model = SentenceTransformer(model_name)
    embeddings = model.encode(texts)

    kmeans = KMeans(n_clusters=n_clusters, random_state=42)
    labels = kmeans.fit_predict(embeddings)

    # Get cluster centers for interpretation
    cluster_centers = kmeans.cluster_centers_

    return labels, cluster_centers

# Usage
texts = ["AI research", "machine learning", "cooking recipes", ...]
labels, centers = cluster_texts(texts, n_clusters=3)
```

---

## Performance Optimization

### 1. Flash Attention (Recommended)

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "Qwen/Qwen3-Embedding-8B",
    model_kwargs={
        "attn_implementation": "flash_attention_2",
        "device_map": "auto"
    },
    tokenizer_kwargs={"padding_side": "left"}
)
```

### 2. Reduced Precision

```python
import torch
from transformers import AutoModel

# Load in float16 (half memory)
model = AutoModel.from_pretrained(
    "Qwen/Qwen3-Embedding-8B",
    torch_dtype=torch.float16,
    device_map="auto"
)
```

### 3. Custom Embedding Dimension (MRL Support)

```python
from sentence_transformers import SentenceTransformer

# Reduce embedding dimension for storage efficiency
# Qwen3-Embedding supports dimensions from 32 to 4096
model = SentenceTransformer("Qwen/Qwen3-Embedding-8B")
embeddings = model.encode(["text"])

# Truncate to lower dimension if needed
reduced_dim = 1024
truncated_embeddings = embeddings[:, :reduced_dim]
```

---

## Integration with project_config.yaml

When using embedding models in experiments, reference model configuration from `project_config.yaml`:

```python
import yaml
from pathlib import Path

def load_embedding_model_from_config():
    """Load embedding model based on project_config.yaml."""
    cfg_path = Path("project_config.yaml")
    if not cfg_path.exists():
        raise FileNotFoundError("project_config.yaml not found")

    cfg = yaml.safe_load(cfg_path.read_text())

    # Find embedding model in config
    for model in cfg.get("models", []):
        if "embedding" in model.get("name", "").lower():
            model_path = model.get("path", model.get("name"))
            return SentenceTransformer(model_path)

    # Default fallback
    return SentenceTransformer("Qwen/Qwen3-Embedding-4B")
```

---

## Best Practices

1. **Use instructions for queries**: Add task-specific instructions to queries for 1-5% performance boost
2. **Batch processing**: Process texts in batches (32-128) for efficiency
3. **Normalize embeddings**: Always normalize before similarity computation
4. **Use appropriate model size**: 0.6B for prototyping, 4B for balanced, 8B for maximum quality
5. **Memory management**: Use `torch.float16` and batch processing for large corpora
6. **Instruction language**: Write instructions in English even for multilingual tasks (training data bias)

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `KeyError: 'qwen3'` | Upgrade transformers to >= 4.51.0 |
| OOM during encoding | Reduce batch size, use `torch.float16`, or switch to smaller model |
| Slow encoding | Enable flash attention, increase batch size, use GPU |
| Poor retrieval quality | Add task-specific instruction to queries |
| Wrong embedding shape | Check pooling strategy (Qwen3 uses last-token pooling) |
| Inconsistent results | Ensure consistent padding side (`left` for Qwen3) |

---

## Reference

- Model card: https://huggingface.co/Qwen/Qwen3-Embedding-8B
- Blog post: https://qwenlm.github.io/blog/qwen3-embedding/
- GitHub: https://github.com/QwenLM/Qwen3-Embedding