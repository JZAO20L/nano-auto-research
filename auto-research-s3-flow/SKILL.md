---
name: auto-research-s3-flow
description: Stage 3 flow control — orchestrate paper writing from frozen idea and experiment results through review loop to final deliverables.
metadata:
  version: "0.1"
---

# Stage 3: Writing & Review — Flow Control

## Goal

Transform a frozen research idea + experiment results into a complete paper draft with figures, tables, and review responses.

**Inputs:** `docs/topic_gap_idea_frozen.md`, `docs/experiment_results.md`, `docs/related_work.md`
**Outputs:** `paper/paper_draft.md`, `paper/review_responses.md`, `paper/figures/`, `paper/tables/`

## Sub-task Sequence

Execute in order. Update `docs/stage3_progress.md` after each step.

### Step 1: Snapshot Frozen Idea

Copy the current frozen idea to `docs/topic_gap_idea_frozen.md` if not already present. This is the single source of truth for the paper's positioning.

### Step 2: Draw Fig.1 — Problem Illustration

Invoke skill: `auto-research-s3-figure-drawio`
- Type: problem/motivation figure
- Output: `paper/figures/fig1_problem.pdf`

### Step 3: Draw Fig.2 — Method Overview

Invoke skill: `auto-research-s3-figure-drawio`
- Type: system architecture figure
- Output: `paper/figures/fig2_method.pdf`

### Step 4: Generate Tables

Invoke skill: `auto-research-s3-table-generator`
- Input: `docs/experiment_results.md`
- Output: `paper/tables/tab_*.tex` + markdown versions

### Step 5: Generate Plots (Optional)

Invoke skill: `auto-research-s3-figure-plot`
- Create plots for: training curves, sensitivity analysis, heatmaps (if data available)
- Output: `paper/figures/*.pdf`

### Step 6: Write Paper Sections

Invoke skill: `auto-research-s3-paper-writing`
- Write all sections in order: Abstract → Introduction → Related Work → Method → Experiments → Conclusion
- Output: `paper/paper_draft.md`

### Step 7: Review Loop

Invoke skills: `auto-research-s3-paper-review` → `auto-research-s3-paper-revision`

**Loop rules:**
- Maximum 3 review rounds
- Stop condition: no major issues remain (score ≥ 7/10)
- Each round: review → revise → re-review

```
for round in 1..3:
    review = run(auto-research-s3-paper-review)
    if no major issues in review:
        break
    run(auto-research-s3-paper-revision)
```

## Rollback Protocol

If review identifies **missing critical experiments** (M1 [major] in Missing Experiments):
1. Do NOT fabricate results
2. Document required experiments in `docs/stage3_progress.md` under "Rollback Required"
3. Report to auto-research orchestrator for Stage 2 supplementation
4. Pause writing until new results arrive in `docs/experiment_results.md`

## Progress Tracking

Maintain `docs/stage3_progress.md`:

```markdown
# Stage 3 Progress

## Status: [in_progress | blocked | complete]
## Current Step: [1-7]
## Review Round: [0-3]

## Completed
- [x] Step 1: Frozen idea snapshot
- [ ] Step 2: Fig.1
...

## Rollback Required
- (none)

## Blockers
- (none)
```

## Final Deliverables Checklist

- [ ] `paper/paper_draft.md` — complete manuscript
- [ ] `paper/review_responses.md` — all review rounds + rebuttals
- [ ] `paper/figures/fig1_problem.pdf`
- [ ] `paper/figures/fig2_method.pdf`
- [ ] `paper/tables/tab_*.tex`
- [ ] `docs/stage3_progress.md` — marked complete
