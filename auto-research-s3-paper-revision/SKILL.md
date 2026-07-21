---
name: auto-research-s3-paper-revision
description: Revise paper draft based on review comments — address issues by priority, write rebuttal responses, maintain document consistency.
metadata:
  version: "0.1"
---

# Stage 3: Paper Revision

## Input

- `paper/review_responses.md` — latest review round (weaknesses, questions, missing experiments)
- `paper/paper_draft.md` — current manuscript to revise

## Output

- Updated `paper/paper_draft.md`
- Rebuttal responses appended to `paper/review_responses.md`

## Revision Process

### Step 1: Triage Comments

Read the latest review round. Categorize and prioritize:

1. **Major weaknesses** — must address (do first)
2. **Missing experiments [major]** — check if data exists in `docs/experiment_results.md`
3. **Minor weaknesses** — address after majors
4. **Questions** — answer in rebuttal, add clarification to paper
5. **Suggestions** — address if straightforward

### Step 2: For Each Comment, Execute

For every issue W/M/Q:

1. **Locate** the relevant section in `paper/paper_draft.md`
2. **Determine action:**
   - `rewrite` — restructure/clarify existing text
   - `add_content` — insert new paragraph/section
   - `add_experiment` — incorporate results from `docs/experiment_results.md`
   - `acknowledge` — add as limitation (when experiment is infeasible)
   - `flag_rollback` — requires new experiments not yet run (see below)
3. **Execute** the change in `paper/paper_draft.md`
4. **Write response** in rebuttal format

### Step 3: Write Rebuttal Responses

Append to `paper/review_responses.md` under the review round:

```markdown
## Author Response — Round N

> W1 [major]: "The paper lacks comparison with method X"

**Response**: We have added method X to Table 1. Results show our method
outperforms X by 12.3% on HarmBench. We believe X was omitted initially
because [reason], but agree it is an important baseline.
**Changes**: Section 5.1, paragraph 2 (added baseline description); Table 1, row 4.

> W2 [minor]: "The notation in Section 3.2 is inconsistent"

**Response**: We have unified notation to use $s_i$ for skills throughout.
**Changes**: Section 3.2, equations 2-4; Section 4.1, paragraph 1.

> M1 [major]: "Transfer experiments across model families are missing"

**Response**: [FLAG: ROLLBACK REQUIRED] This experiment requires running
attacks on ModelX which is not in current results. Flagged for Stage 2.
**Changes**: None pending new results.
```

### Step 4: Consistency Check

After all revisions, verify:

- [ ] Abstract matches final method description and results
- [ ] Introduction contributions list matches what experiments show
- [ ] Method section matches Fig.2 (no orphaned components)
- [ ] All tables/figures are referenced in text
- [ ] Terminology is consistent throughout
- [ ] Conclusion reflects actual limitations discovered during review

## Rollback Protocol

If a major issue requires **new experiments not in `docs/experiment_results.md`**:

1. Do NOT fabricate or estimate results
2. Mark the response as `[FLAG: ROLLBACK REQUIRED]`
3. Add to `docs/stage3_progress.md` under "Rollback Required":
   ```markdown
   - [ ] Run [specific experiment] for [review comment W/M number]
   ```
4. Report to auto-research orchestrator for Stage 2 supplementation
5. Continue revising other comments that don't depend on missing data

## Priority Rules

- Address ALL major issues (no exceptions)
- Address ALL questions (reviewers expect answers)
- Address minor issues when they don't conflict with majors
- Suggestions: only if they improve clarity without expanding scope
- If two comments conflict, prefer the one with higher severity

## Steps

1. Read latest review round from `paper/review_responses.md`
2. Sort comments: major → minor → suggestion
3. For each comment: locate → decide action → edit paper → write response
4. Run consistency check (Step 4 above)
5. Append all responses to `paper/review_responses.md`
6. Update `docs/stage3_progress.md` with revision status
7. Report summary: N major fixed, M minor fixed, K flagged for rollback
