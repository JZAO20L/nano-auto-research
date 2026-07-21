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

## 1. Project Initialization

On first invocation, create the project directory structure:

```
PROJECT_ROOT/
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
- **Topic**: {user_topic}
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
