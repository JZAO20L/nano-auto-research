---
name: auto-research-s1-flow
description: Stage 1 flow control — from user topic to literature review, gap analysis, idea proposal, and asset preparation. Manages the iterative search loop and completion criteria.
metadata:
  version: "0.1"
---

# S1 Flow: Deep Research

**Stage goal**: From user's research topic → produce `related_work.md`, `topic_gap_idea.md`, `assets.md`, `baselines.md`.

## 1. Search Loop

Iterate the following cycle (max **10 rounds**):

### Step 1: Design Keywords
Based on current gap understanding, generate 3–5 search keyword combinations.
- Round 1: broad topic terms (e.g., "LLM jailbreak attack", "prompt injection defense")
- Later rounds: refine based on gaps found (e.g., "transferable adversarial suffix multi-model")

### Step 2: Search
For each keyword set, invoke `auto-research-s1-arxiv-search` to retrieve papers.
Also search GitHub (`auto-research-s1-github-search`) for baseline repos when a relevant method is found.

> **Skill invocation**: To invoke a sub-skill, read its `SKILL.md` file and follow the instructions within it. Skills are guidance documents, not executable commands.

### Step 3: Analyze & Reflect
For highly relevant papers (method match + recent), invoke `auto-research-s1-paper-analysis` for deep analysis.
Append structured entries to `docs/related_work.md`.

### Step 4: Update Gap
Update `docs/topic_gap_idea.md` with:
- What has been explored (covered areas)
- What remains unexplored (gaps)
- Emerging patterns or contradictions across papers

### Step 5: Refine Keywords
Based on gaps, adjust keyword strategy for next round.

## 2. Completion Assessment

After each round, evaluate 4 criteria:

| # | Criterion | Check |
|---|-----------|-------|
| 1 | **Gap stable** | Gap description unchanged for 2 consecutive rounds |
| 2 | **Search saturated** | New rounds yield <2 new relevant papers |
| 3 | **Routes covered** | All major method families in the topic are represented |
| 4 | **Positioning writable** | Can articulate "existing work does X, we propose Y because Z" |

**Pass condition**: 3/4 criteria met, OR max rounds (10) reached.

Log assessment in `docs/stage1_progress.md`:
```markdown
## Round {N} Assessment
- Gap stable: ✓/✗
- Search saturated: ✓/✗
- Routes covered: ✓/✗
- Positioning writable: ✓/✗
- Decision: continue / complete
```

## 3. Positioning & Idea Proposal

After search completes:

1. Write a positioning statement in `topic_gap_idea.md`:
   ```
   Existing work focuses on [X]. However, [gap]. We propose to [Y] by [key insight].
   ```

2. Propose **3–5 research ideas**, each with:
   - **Title**: one-line description
   - **Method sketch**: 2–3 sentences on approach
   - **Novelty**: what's new vs. closest existing work
   - **Feasibility**: compute/data/timeline estimate (high/medium/low)
   - **Risk**: what could go wrong
   - **Score**: novelty(1-5) × feasibility(1-5)

3. Rank ideas by score. Present to user as decision gate.

## 4. Asset Preparation

After user confirms an idea, prepare `docs/assets.md` and `docs/baselines.md`:

### assets.md format:
```markdown
## Models
| Model | Platform | ID | Download Command | Status |
|-------|----------|----|-----------------|--------|
| Qwen2.5-7B | ModelScope | Qwen/Qwen2.5-7B-Instruct | `modelscope download ...` | pending |

## Datasets
| Dataset | Platform | ID | Download Command | Status |
|---------|----------|----|-----------------|--------|
```

### baselines.md format:
```markdown
| Method | Paper | Repo | Stars | Reproduction | Status |
|--------|-------|------|-------|--------------|--------|
| GCG | arxiv:2307.15043 | llm-attacks/llm-attacks | 800 | ready | pending |
```

Use `auto-research-s1-modelscope-query` and `auto-research-s1-huggingface-query` to verify model/dataset availability.
Use `auto-research-s1-github-search` to find baseline repos.

Execute downloads. Update status in assets.md/baselines.md.

## 5. Progress Tracking

Maintain `docs/stage1_progress.md`:
```markdown
# Stage 1 Progress
- **Topic**: {topic}
- **Rounds completed**: {N}/10
- **Papers analyzed**: {count}
- **Phase**: search_loop | idea_proposal | asset_prep | gate_pending
- **Last updated**: {date}
```

## 6. Decision Gate

Present to user:
1. Summary of literature landscape (key method families, coverage)
2. Identified gap and positioning
3. Ranked idea list with scores
4. Asset/baseline readiness status

**Wait for user to select an idea before proceeding to S2.**
