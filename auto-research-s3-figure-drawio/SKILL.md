---
name: auto-research-s3-figure-drawio
description: Create academic figures (problem illustration and method overview) using drawio XML or structured text descriptions.
metadata:
  version: "0.1"
---

# Stage 3: Drawio Figure Creation

## Figure Types

### Fig.1 — Problem Illustration (Motivation)

**Purpose:** Visually show the research gap. Reader should understand "why this problem matters" in 5 seconds.

**Structure:** "Current approach → Limitation → Our insight"

**Constraints:**
- 4–6 nodes maximum
- No formulas, no implementation details
- One clear visual flow (left-to-right or top-to-bottom)

**Example (jailbreak research):**
```
[LLM with Safety Alignment] → [Jailbreak Attack] → [Harmful Output]
                                    ↓
                          [Existing: single-shot, no adaptation]
                                    ↓
                          [Our insight: self-evolving skills]
```

### Fig.2 — Method Overview (Architecture)

**Purpose:** Show the full system at a glance. Reader should understand component relationships.

**Structure:** Input → Processing components → Output

**Constraints:**
- 6–10 nodes maximum
- Show data flow with labeled arrows
- Group related components with a container/box
- No formulas in the overview (details go in Method section text)

**Example:**
```
[Input Queries] → [Skill Library] → [Skill Selection]
                       ↑                    ↓
              [Skill Evolution] ← [Reward Evaluation] ← [Target LLM Response]
```

## Design Principles

1. **Simplified and clean** — fewer nodes, core structure + data flow only
2. **No implementation details** — no class names, no hyperparameters, no code
3. **No formulas** — use descriptive labels instead (e.g., "reward signal" not "r = f(x,y)")
4. **Blue-based academic palette:**
   - Primary boxes: `#4472C4` (fill), white text
   - Secondary boxes: `#D6E4F0` (fill), dark text
   - Arrows/flow: `#2F5496`
   - Highlight/ours: `#ED7D31` (orange accent, sparingly)
5. **Consistent sizing** — all boxes same height, aligned to grid

## Drawio XML Template

```xml
<mxGraphModel>
  <root>
    <mxCell id="0"/>
    <mxCell id="1" parent="0"/>
    <!-- Node -->
    <mxCell id="2" value="Skill Library"
      style="rounded=1;whiteSpace=wrap;fillColor=#4472C4;fontColor=#ffffff;strokeColor=#2F5496;"
      vertex="1" parent="1">
      <mxGeometry x="100" y="100" width="140" height="50" as="geometry"/>
    </mxCell>
    <!-- Edge -->
    <mxCell id="3" value="select"
      style="edgeStyle=orthogonalEdgeStyle;strokeColor=#2F5496;"
      edge="1" source="2" target="4" parent="1">
      <mxGeometry relative="1" as="geometry"/>
    </mxCell>
  </root>
</mxGraphModel>
```

## Alternative: Structured Text Description

If drawio XML is impractical, output a structured description for manual drawing:

```markdown
## Fig.1: Problem Illustration

Nodes:
- A: "Safety-Aligned LLM" (primary, x=0, y=0)
- B: "Jailbreak Attack" (secondary, x=1, y=0)
- C: "Harmful Output" (highlight, x=2, y=0)
- D: "Limitation: static, single-shot" (annotation, x=1, y=1)

Edges:
- A → B: "prompt"
- B → C: "bypasses alignment"
- B → D: (dashed, no label)
```

## Output

- `paper/figures/fig1_problem.pdf` (or `fig1_problem.drawio`)
- `paper/figures/fig2_method.pdf` (or `fig2_method.drawio`)

## Steps

1. Read `docs/topic_gap_idea_frozen.md` for problem statement and method description
2. Identify core nodes (what are the 4-6 key concepts for Fig.1? 6-10 for Fig.2?)
3. Determine layout direction (left-to-right for flow, top-to-bottom for hierarchy)
4. Generate drawio XML or structured text
5. Save to `paper/figures/`
6. Verify: node count within limits, no formulas, blue palette applied
