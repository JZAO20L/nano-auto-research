---
name: auto-research-s3-paper-review
description: Simulate rigorous peer review of the paper draft — produce structured reviews with severity-tagged weaknesses and missing experiments.
metadata:
  version: "0.1"
---

# Stage 3: Simulated Peer Review

## Mindset

Switch to **reviewer perspective**. Read `paper/paper_draft.md` as if seeing it for the first time at a top venue (AAAI/ACL/NeurIPS). Be critical but constructive.

## Review Dimensions

Evaluate along these axes:

1. **Novelty** — Is the contribution clearly new vs. prior work?
2. **Technical soundness** — Is the method well-defined? Are claims supported?
3. **Experimental completeness** — Enough baselines, benchmarks, ablations?
4. **Clarity** — Can a reader reproduce the method from the paper alone?
5. **Significance** — Does this advance the field meaningfully?

## Output Format

Append to `paper/review_responses.md`:

```markdown
---
# Review Round N
Date: YYYY-MM-DD

## Summary
(2-3 sentences describing what the paper does and its main contribution)

## Strengths
- S1: [specific strength with section/table reference]
- S2: ...
- S3: ...

## Weaknesses
- W1 [major]: [specific issue, reference section/line]
- W2 [minor]: [specific issue]
- W3 [suggestion]: [improvement idea]

## Questions
- Q1: [specific question the authors should answer]
- Q2: ...

## Missing Experiments
- M1 [major]: [experiment needed, why it matters]
- M2 [minor]: [nice-to-have experiment]

## Overall Assessment
Score: X/10
Confidence: X/5
Summary judgment: [accept/borderline/reject and one-sentence reason]
```

## Severity Definitions

| Severity | Meaning | Action Required |
|----------|---------|-----------------|
| **major** | Would reject if unaddressed | Must fix before next round |
| **minor** | Should fix, not rejection-worthy | Fix if time permits |
| **suggestion** | Nice-to-have | Optional improvement |

### Major examples:
- Missing critical baseline comparison
- Flawed experimental logic (metric doesn't measure what's claimed)
- Unclear contribution (reader cannot distinguish from prior work)
- Missing ablation for a key claimed component

### Minor examples:
- Unclear paragraph or ambiguous notation
- Missing citation for a claim
- Inconsistent terminology
- Figure/table not referenced in text

### Suggestion examples:
- Additional visualization (e.g., qualitative examples)
- Broader impact discussion
- Comparison on additional benchmark

## Cross-check Procedure

Before finalizing the review:

1. Read `docs/pre_review_checklist.md` (if exists)
2. For each predicted reviewer question in the checklist, verify the paper addresses it
3. If a predicted question is NOT answered in the paper, add it as a Weakness or Question

## Review Quality Checks

- Every weakness must reference a specific section/paragraph/table
- Every "missing experiment" must explain WHY it's needed
- Strengths should be genuine (don't pad) — 2-4 is normal
- Score calibration: 7+ = accept-ready, 5-6 = borderline, <5 = reject
- Do NOT suggest experiments that are infeasible (e.g., requiring 100x compute)

## Steps

1. Read `paper/paper_draft.md` completely (one pass for understanding)
2. Second pass: evaluate each dimension, note issues
3. Read `docs/pre_review_checklist.md`, cross-check
4. Write structured review in the format above
5. Assign score and confidence
6. Append to `paper/review_responses.md` with round number
7. Report: count of major/minor/suggestion issues for flow control
