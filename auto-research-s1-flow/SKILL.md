---
name: auto-research-s1-flow
description: Stage 1 flow control — from user topic to literature review, gap analysis, idea proposal, and asset preparation. Manages the iterative search loop and completion criteria.
metadata:
  version: "0.1"
---

# S1 Flow: Deep Research

**Stage goal**: From user's research topic → produce `related_work.md`, `topic_gap_idea.md`, `assets.md`, `baselines.md`.

## 1. Search Loop

Iterate the following cycle. **Termination**: round > 5 AND new papers found in current round ≤ 5.

> **Skill invocation**: To invoke a sub-skill, read its `SKILL.md` file and follow the instructions within it. Skills are guidance documents, not executable commands.

### Step 1: Design Keywords
Generate 3–5 search keyword combinations based on **all prior context**:
- Round 1: broad topic terms (e.g., "LLM jailbreak attack", "prompt injection defense")
- Later rounds: derive from current GAP analysis and idea pool — target under-explored directions, avoid re-searching saturated areas (e.g., if GAP says "no cross-architecture skill transfer", search "transferable adversarial strategy multi-model")

### Step 2: Search
For each keyword set, invoke `auto-research-s1-arxiv-search` to retrieve papers.
Also search GitHub (`auto-research-s1-github-search`) for baseline repos when a relevant method is found.
**Count new papers** (not previously seen in prior rounds).

### Step 3: Cumulative Synthesis
This is the core of each round. Update ALL THREE documents:

1. **`docs/related_work.md`** — append new paper entries (structured: title, venue, method, key results, relevance)
2. **`docs/topic_gap_idea.md` § Gap Analysis** — update:
   - Which directions are now well-covered (with evidence: paper count, key works)
   - Which gaps remain or newly emerged
   - Contradictions or open questions across papers
3. **`docs/topic_gap_idea.md` § Idea Pool** — update:
   - For each existing idea: note overlap with newly found papers (mark `⚠️ overlap` if a paper already does this)
   - Add new ideas inspired by this round's findings
   - This prevents proposing ideas that existing work already covers

### Step 4: Plan Next Round
Based on the updated GAP and idea pool:
- Which gap directions need deeper search?
- Which keyword angles are exhausted (skip next round)?
- Any new method families discovered that need coverage?

### Step 5: Check Termination
- If round > 5 AND new papers this round ≤ 5 → **stop searching** (minimum rounds done + diminishing returns)
- Otherwise → continue to next round

Log in `docs/stage1_progress.md`:
```markdown
## Round {N}
- Keywords: [list]
- New papers found: {count}
- GAP update: {1-line summary of what changed}
- Idea pool: {added/removed/flagged overlap}
- Decision: continue / terminate (reason)
```

## 2. Positioning & Idea Finalization

After search terminates, finalize the idea pool built during the search loop:

1. Write a positioning statement in `topic_gap_idea.md`:
   ```
   Existing work focuses on [X]. However, [gap]. We propose to [Y] by [key insight].
   ```

2. Review the idea pool — remove ideas flagged `⚠️ overlap`, then for remaining **3–5 ideas** finalize:
   - **Title**: one-line description
   - **Method sketch**: 2–3 sentences on approach
   - **Novelty**: what's new vs. closest existing work (cite specific papers from related_work.md)
   - **Feasibility**: compute/data/timeline estimate (high/medium/low)
   - **Risk**: what could go wrong
   - **Score**: novelty(1-5) × feasibility(1-5)

3. Rank ideas by score. Present to user as decision gate.

## 3. Asset Preparation

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

## 4. Progress Tracking

Maintain `docs/stage1_progress.md`:
```markdown
# Stage 1 Progress
- **Topic**: {topic}
- **Rounds completed**: {N}/5
- **Papers analyzed**: {count}
- **Phase**: search_loop | idea_finalization | asset_prep | gate_pending
- **Last updated**: {date}
```

## 5. Decision Gate

Present to user:
1. Summary of literature landscape (key method families, coverage)
2. Identified gap and positioning
3. Ranked idea list with scores
4. Asset/baseline readiness status

**Wait for user to select an idea before proceeding to S2.**
