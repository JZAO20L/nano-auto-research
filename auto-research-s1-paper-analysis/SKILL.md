---
name: auto-research-s1-paper-analysis
description: Guide for deep analysis of a single academic paper. Extracts structured information for related_work.md including method, experiments, limitations, and relevance.
metadata:
  version: "0.1"
---

# S1 Paper Analysis

Deep-analyze a single paper and produce a structured entry for `docs/related_work.md`.

## 1. When to Deep-Analyze

Apply deep analysis when a paper meets BOTH:
- **Highly relevant**: proposes a method directly related to our research direction
- **Recent**: published within the last 2 years

Papers that are only tangentially related or older → add a brief 2-line entry to related_work.md without deep analysis.

## 2. Obtain Full Text

Use the arxiv search skill (`auto-research-s1-arxiv-search`) to get the paper ID, then download HTML:

```bash
# Preferred: HTML version (better parsing)
curl -s -L "https://arxiv.org/html/{PAPER_ID}" -o /tmp/paper_{PAPER_ID}.html

# Fallback: abstract page if HTML unavailable
curl -s "https://arxiv.org/abs/{PAPER_ID}" -o /tmp/paper_{PAPER_ID}.html
```

If HTML is unavailable, use the abstract + any information from the search results.

## 3. Extraction Template

Extract the following from the paper:

```markdown
### {Paper Title} ({Year})
- **Authors**: {first author} et al.
- **Venue**: {conference/journal}
- **arXiv**: {id}

**Problem**: What specific problem does this paper address? (1-2 sentences)

**Method**: Key components of the proposed approach:
1. {component 1}
2. {component 2}
3. {component 3}

**Experiments**:
- Datasets: {list}
- Metrics: {list}
- Baselines compared: {list}
- Main result: {key number/finding}

**Limitations**: Acknowledged or inferred weaknesses:
- {limitation 1}
- {limitation 2}

**Relevance to our work**: How does this relate to our research direction?
- Similarity: {what we share}
- Difference: {what we do differently}
- Gap it reveals: {what's still missing}
```

## 4. Output

Append the structured entry to `docs/related_work.md` under the appropriate method-family section.

If the paper reveals a new method family not yet in related_work.md, create a new section.

## 5. Focus Areas

When analyzing, prioritize:

1. **Method novelty**: What is genuinely new vs. incremental combination of existing techniques?
2. **Experimental reproducibility**: Are datasets public? Is code available? Are hyperparameters specified?
3. **Claimed vs. actual contributions**: Do the experiments actually support the claims? Are baselines fair?
4. **Transferability**: Can the method generalize beyond the specific setting tested?

## 6. Red Flags to Note

Flag in the entry if you observe:
- Evaluation on non-standard or private datasets only
- Missing comparison with obvious baselines
- Overclaimed novelty (method is essentially X with minor modification)
- Results that don't support the stated conclusion
