---
name: auto-research-s3-table-generator
description: Generate LaTeX and Markdown tables from experiment results — main results, ablation, and efficiency comparisons.
metadata:
  version: "0.1"
---

# Stage 3: Table Generation

## Input / Output

- **Input:** `docs/experiment_results.md` (markdown tables with numerical results)
- **Output:** `paper/tables/tab_NAME.tex` (LaTeX) + markdown version embedded in `paper/paper_draft.md`

## LaTeX Table Template — Main Results

**Required LaTeX packages**: `booktabs` (for `\toprule`, `\midrule`, `\bottomrule`), `multirow` (if using multi-row headers). Add to preamble:
```latex
\usepackage{booktabs}
\usepackage{multirow}
```

```latex
\begin{table}[t]
\centering
\caption{Attack success rate (\%) on three benchmarks. Best results in \textbf{bold}, second-best \underline{underlined}.}
\label{tab:main_results}
\begin{tabular}{l|ccc}
\toprule
Method & HarmBench & AdvBench & JailbreakBench \\
\midrule
GCG & 42.1 & 38.5 & 35.2 \\
PAIR & 51.3 & 47.8 & 44.6 \\
AutoDAN & 55.7 & 52.1 & 48.3 \\
\midrule
Ours & \textbf{67.3} & \textbf{63.8} & \textbf{60.1} \\
\bottomrule
\end{tabular}
\end{table}
```

## Formatting Rules

1. **Bold** the best result in each column: `\textbf{67.3}`
2. **Underline** the second-best: `\underline{55.7}`
3. Use `\toprule`, `\midrule`, `\bottomrule` (booktabs package)
4. Separate baselines from "Ours" with a `\midrule`
5. Caption describes what is measured + bold/underline convention
6. Always include `\label{tab:...}` for cross-referencing

## Ablation Table Template

```latex
\begin{table}[t]
\centering
\caption{Ablation study. $\Delta$ shows change from full model.}
\label{tab:ablation}
\begin{tabular}{l|cc|c}
\toprule
Variant & HarmBench & AdvBench & $\Delta$ \\
\midrule
Full model & \textbf{67.3} & \textbf{63.8} & -- \\
\midrule
w/o skill evolution & 58.1 & 55.2 & $-$9.2 \\
w/o skill selection & 61.4 & 58.7 & $-$5.9 \\
w/o reward shaping & 63.0 & 60.1 & $-$4.3 \\
\bottomrule
\end{tabular}
\end{table}
```

## Efficiency Table Template

```latex
\begin{table}[t]
\centering
\caption{Computational cost comparison.}
\label{tab:efficiency}
\begin{tabular}{l|ccc}
\toprule
Method & Queries & Time (min) & GPU Mem (GB) \\
\midrule
GCG & 500 & 12.3 & 24.1 \\
PAIR & 200 & 8.7 & 16.5 \\
Ours & \textbf{150} & \textbf{6.2} & 16.8 \\
\bottomrule
\end{tabular}
\end{table}
```

## Markdown Version (for paper_draft.md)

```markdown
| Method | HarmBench | AdvBench | JailbreakBench |
|--------|-----------|----------|----------------|
| GCG | 42.1 | 38.5 | 35.2 |
| PAIR | 51.3 | 47.8 | 44.6 |
| **Ours** | **67.3** | **63.8** | **60.1** |
```

## Steps

1. Read `docs/experiment_results.md` — identify all result tables
2. For each table, determine type: main results / ablation / efficiency
3. Extract numbers, identify best and second-best per column
4. Generate LaTeX with proper formatting (bold, underline, midrules)
5. Save to `paper/tables/tab_NAME.tex`
6. Generate markdown version for inclusion in `paper/paper_draft.md`
7. Verify: all numbers match source, bold/underline correct, labels unique

## Naming Convention

- `tab_main_results.tex` — primary comparison table
- `tab_ablation.tex` — component removal study
- `tab_efficiency.tex` — computational cost
- `tab_transfer.tex` — cross-model transfer results
- `tab_sensitivity.tex` — hyperparameter sensitivity
