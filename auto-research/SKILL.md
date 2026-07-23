---
name: auto-research
description: Top-level orchestrator for the 3-stage auto-research pipeline (Deep Research → Experimenting → Writing). Manages project lifecycle, stage dispatch, decision gates, and rollback.
metadata:
  version: "0.1"
---

# Auto-Research Orchestrator

You are the top-level controller of a 3-stage research automation pipeline:
- **S1 (Deep Research)**: topic → literature review → research idea selection
- **S2 (Experimenting)**: idea → implementation → experiments → results
- **S3 (Writing)**: results → paper draft

```mermaid
flowchart TD
    START([用户启动]) --> INIT[Project Init<br/>校验 project_config.yaml]
    INIT --> S1[S1: Deep Research<br/>文献搜索 → GAP → Idea]
    S1 --> G1{用户选定 Idea?}
    G1 -->|确认| S2[S2: Coding & Experimenting<br/>实现 → 实验 → 结果]
    G1 -->|换方向| S1
    S2 --> G2{结果充分?}
    G2 -->|确认| S3[S3: Writing & Review<br/>写作 → 审稿 → 修改]
    G2 -->|补实验| S2
    S3 --> G3{审稿通过?}
    G3 -->|通过| DONE([完成])
    G3 -->|缺实验| S2
    G3 -->|修改| S3
```

## 1. Project Initialization

### 1.1 Check for project_config.yaml

On first invocation, check if `project_config.yaml` exists in the project root.

**If NOT present**: present the user with the template below and ask them to fill it in. Do NOT proceed until the config file exists.

```yaml
# project_config.yaml — 用户填写，agent 只读
topic: "Research topic description"
method_sketch: "Brief description of the planned approach (optional but recommended)"
target_venue: "AAAI 2027"
constraints:
  compute: "8x A100 80GB"
  timeline: "3 months"

models:
  # Local models: agent checks path, downloads if missing
  - name: Qwen3-4B
    type: local
    path: /data/models/Qwen3-4B
    description: "Backbone for GRPO training"

  # API models: agent tests connectivity, respects rate limits
  - name: qwen-max
    type: api
    url: https://dashscope.aliyuncs.com/compatible-mode/v1
    api_key: sk-xxx
    max_concurrency: 10
    rpm: 60
    description: "Remote baseline"
```

**If present**: validate required fields:
1. `topic` must be non-empty
2. `models` must have at least 1 entry
3. Each model must have `name`, `type`, and `description`
4. Local models (`type: local`) must have `path`
5. API models (`type: api`) must have `url`, `api_key`, `max_concurrency`, `rpm`

If validation fails, report the specific missing/invalid fields and ask the user to fix `project_config.yaml`. Do NOT proceed until validation passes.

### 1.2 Create Project Directory Structure

After validation, create the project directory structure:

```bash
mkdir -p projects/{name}/{docs,data,models,baselines,src,scripts,exp,output,paper}
```

```
PROJECT_ROOT/
├── project_config.yaml          # user-provided config (agent read-only)
├── docs/
│   ├── project_status.md        # pipeline state (you maintain this)
│   ├── stage1_progress.md       # S1 search progress
│   ├── related_work.md          # literature review
│   ├── topic_gap_idea.md        # gap analysis + idea pool (living doc)
│   ├── assets.md                # models/datasets to download
│   └── baselines.md             # baseline methods + repos
├── src/                         # experiment code
├── exp/                         # experiment outputs
├── paper/                       # LaTeX draft
└── assets/                      # downloaded models/data
    ├── models/
    └── data/
```

Initialize `docs/project_status.md`:

```markdown
# Project Status
- **Topic**: {topic from project_config.yaml}
- **Target Venue**: {target_venue from project_config.yaml}
- **Current Stage**: S1
- **Stage Phase**: init
- **Last Updated**: {date}
- **Active Idea**: (none)
- **Rollback Count**: 0
```

## 2. Stage Dispatch

Read `docs/project_status.md` to determine current stage and phase. Load the corresponding skill:

| Stage | Skill to Load |
|-------|--------------|
| S1    | `auto-research-s1-flow` |
| S2    | `auto-research-s2-flow` |
| S3    | `auto-research-s3-flow` |

After the stage skill completes its work, return here for the decision gate.

## 3. Decision Gates

At each stage boundary, **pause and present to user**:

- **S1→S2**: Present ranked idea list with feasibility/novelty scores. Ask user to select idea. Freeze `topic_gap_idea.md` (rename to `topic_gap_idea_frozen.md`, create new living copy for potential rollback).
- **S2→S3**: Present experiment results summary table. Ask user if results are sufficient for writing.
- **S3→Done**: Present paper draft. Ask user for review.

Do NOT proceed past a gate without explicit user confirmation.

## 4. Rollback Management

| Rollback | Trigger | Action |
|----------|---------|--------|
| S2→S2 | Current idea fails experiments | Switch to next idea in frozen pool, reset S2 phase |
| S2→S1 | All ideas in pool exhausted | Unfreeze `topic_gap_idea.md`, re-enter S1 search with refined keywords |
| S3→S2 | Writing reveals missing experiments | Return to S2 with specific experiment requests |

On rollback, update `docs/project_status.md`:
```markdown
- **Rollback Count**: {increment}
- **Rollback Reason**: {reason}
```

Max rollback count: 3. If exceeded, present situation to user for guidance.

## 5. Version Management: topic_gap_idea.md

- **Living state** (during S1): continuously updated as search progresses. Contains gap analysis, candidate ideas with scores.
- **Frozen state** (S1→S2 gate passed): ideas are locked. The selected idea is marked. File renamed to `topic_gap_idea_frozen.md`.
- **Unfrozen** (S2→S1 rollback): frozen file restored to living state, new search round appended.

## 6. Maintaining project_status.md

Update after every significant action:
- Stage/phase transitions
- Idea selection or switch
- Rollback events
- Asset download completion
- Experiment completion milestones

Format for phase tracking:
```markdown
- **Current Stage**: S1
- **Stage Phase**: search_loop | idea_proposal | asset_prep | gate_pending
```
