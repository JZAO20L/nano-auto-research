---
name: auto-research-s3-paper-writing
description: Write complete paper sections in order — Abstract through Conclusion — from frozen idea, related work, and experiment results.
metadata:
  version: "0.1"
---

# Stage 3: Paper Section Writing

## Input Documents

- `docs/topic_gap_idea_frozen.md` → Introduction, Method positioning
- `docs/related_work.md` → Related Work section
- `docs/experiment_results.md` → Experiments section
- `paper/figures/`, `paper/tables/` → references within text

## Output

`paper/paper_draft.md` — complete manuscript in markdown (target: 8 pages main content, AAAI/ACL format)

## Section Order and Content

### 1. Abstract (150–250 words)

Structure: Problem → Gap → Method → Key Results → Impact

```markdown
## Abstract

Large language models (LLMs) are vulnerable to jailbreak attacks that bypass
safety alignment. Existing methods [limitation: static, single-shot, no adaptation].
We propose [METHOD NAME], a [one-sentence description]. Specifically, [key mechanism
in 1-2 sentences]. Experiments on [N] benchmarks across [M] models demonstrate
that [METHOD] achieves [X%] improvement over the strongest baseline. [Broader
implication sentence.]
```

### 2. Introduction (1–1.5 pages)

Structure: Importance → Existing limitations (2-3) → Our approach → Contributions

End with a contributions list:

```markdown
Our contributions are:
- We identify [specific gap/problem] that limits existing approaches.
- We propose [METHOD], which [key innovation in one sentence].
- We conduct extensive experiments on [benchmarks/models], demonstrating
  [summary of results].
- [Optional: resource/dataset/code contribution]
```

Reference Fig.1 here: "As illustrated in Figure 1, ..."

### 3. Related Work (1 page)

2–3 subsections by theme. Each subsection:
- Survey 4–6 relevant works
- End with soft positioning: "Unlike [X], our approach [differentiator]."

```markdown
## 2 Related Work

### 2.1 Jailbreak Attacks on LLMs
[Survey gradient-based, query-based, prompt-based attacks...]
Unlike these single-shot methods, our approach evolves attack strategies iteratively.

### 2.2 Self-Improvement in AI Systems
[Survey self-play, self-refine, evolutionary methods...]
Unlike general self-improvement, we specifically target adversarial skill accumulation.

### 2.3 LLM Safety and Alignment
[Brief survey of defense/alignment work...]
```

### 4. Method (2–2.5 pages)

Structure:
1. Overview paragraph + reference Fig.2: "Figure 2 illustrates the framework of [METHOD]."
2. Problem formulation (notation, objective)
3. Component descriptions (one subsection per major component)
4. Algorithm pseudocode (if applicable)

**Tense:** Present tense for method description ("The module selects...", "We define...")

### 5. Experiments (2–2.5 pages)

Structure:
1. **Setup:** models, datasets, metrics, baselines, implementation details
2. **Main Results:** reference Table 1, compare with baselines, highlight improvements
3. **Ablation Study:** reference Table 2, analyze each component's contribution
4. **Analysis:** transfer experiments, sensitivity, qualitative examples, efficiency

**Tense:** Past tense for experiments ("We evaluated...", "Results showed...")

```markdown
## 5.1 Experimental Setup

**Models.** We evaluate on [list models with sizes].
**Datasets.** We use [list benchmarks with sizes and descriptions].
**Metrics.** Attack Success Rate (ASR) measured by [description].
**Baselines.** We compare with [list methods with citations].
```

### 6. Conclusion (0.5 page)

Structure: Summary of contributions → Limitations → Future work

```markdown
## 6 Conclusion

We presented [METHOD], which [one-sentence summary]. Experiments demonstrate
[key finding]. Limitations include [honest limitation]. Future work includes
[1-2 directions].
```

## Writing Style Rules

- Academic English, formal register
- Present tense for method description, past tense for experiments
- Citation format: [Author et al., Year]
- Avoid: "very", "really", "a lot of", "it is worth noting that"
- Prefer active voice: "We propose..." not "It is proposed..."
- Every claim needs evidence: citation or experimental result

## Steps

1. Read all input documents thoroughly
2. Write sections in order (Abstract last if needed, but include placeholder)
3. Insert figure/table references at appropriate locations
4. Ensure contributions list matches what experiments actually demonstrate
5. Self-check: does each section flow logically to the next?
6. Save complete draft to `paper/paper_draft.md`
