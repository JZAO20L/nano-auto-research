---
name: auto-research-s3-figure-plot
description: Generate publication-ready data visualizations (bar charts, line plots, heatmaps) with matplotlib for ML research papers.
metadata:
  version: "0.1"
---

# Stage 3: Data Visualization with Matplotlib

## When to Use

- Main results comparison across benchmarks → grouped bar chart
- Training curves, sensitivity analysis → line plot
- Cross-model × cross-dataset matrix → heatmap
- Only create plots that add information beyond tables

## Global Style

Apply at the top of every plotting script:

```python
import matplotlib.pyplot as plt
import numpy as np

plt.rcParams.update({
    'font.size': 12,
    'axes.labelsize': 14,
    'axes.titlesize': 14,
    'legend.fontsize': 11,
    'figure.dpi': 300,
    'savefig.bbox': 'tight',
    'savefig.pad_inches': 0.05,
    'axes.spines.top': False,
    'axes.spines.right': False,
})

# Blue-based academic palette
COLORS = ['#4472C4', '#ED7D31', '#A5A5A5', '#FFC000', '#5B9BD5', '#70AD47']
```

## Grouped Bar Chart (Main Results)

```python
methods = ['Baseline1', 'Baseline2', 'Ours']
benchmarks = ['HarmBench', 'AdvBench', 'JailbreakBench']
# data[benchmark] = [score_method1, score_method2, score_ours]
data = {
    'HarmBench': [45.2, 52.1, 67.3],
    'AdvBench': [41.0, 48.5, 63.8],
    'JailbreakBench': [38.7, 44.2, 60.1],
}

x = np.arange(len(benchmarks))
width = 0.25
fig, ax = plt.subplots(figsize=(8, 4.5))

for i, method in enumerate(methods):
    values = [data[b][i] for b in benchmarks]
    bars = ax.bar(x + i * width, values, width, label=method, color=COLORS[i])

ax.set_xticks(x + width)
ax.set_xticklabels(benchmarks)
ax.set_ylabel('Attack Success Rate (%)')
ax.legend(loc='upper left')
ax.set_ylim(0, 100)

plt.savefig('paper/figures/main_results.pdf')
plt.close()
```

## Line Plot (Sensitivity / Training Curve)

```python
steps = [0, 1, 2, 3, 4, 5]
scores_ours = [45, 52, 58, 63, 66, 67]
scores_base = [45, 46, 47, 47, 48, 48]

fig, ax = plt.subplots(figsize=(6, 4))
ax.plot(steps, scores_ours, 'o-', color=COLORS[0], label='Ours', linewidth=2)
ax.plot(steps, scores_base, 's--', color=COLORS[2], label='Baseline', linewidth=2)

ax.set_xlabel('Evolution Round')
ax.set_ylabel('Attack Success Rate (%)')
ax.legend()
ax.grid(True, alpha=0.3)

plt.savefig('paper/figures/sensitivity.pdf')
plt.close()
```

## Heatmap (Cross-model Transfer)

```python
import seaborn as sns

# rows = source models, cols = target models
transfer_matrix = np.array([
    [85.2, 62.1, 58.3, 55.0],
    [60.5, 82.7, 56.1, 53.2],
    [57.8, 55.3, 80.4, 51.8],
    [54.1, 52.0, 50.5, 78.9],
])
models = ['ModelA', 'ModelB', 'ModelC', 'ModelD']

fig, ax = plt.subplots(figsize=(6, 5))
sns.heatmap(transfer_matrix, annot=True, fmt='.1f', cmap='Blues',
            xticklabels=models, yticklabels=models, ax=ax,
            vmin=40, vmax=90, linewidths=0.5)
ax.set_xlabel('Target Model')
ax.set_ylabel('Source Model')
ax.set_title('Transfer Attack Success Rate (%)')

plt.savefig('paper/figures/transfer_heatmap.pdf')
plt.close()
```

## Guidelines

1. **No chartjunk** — no 3D effects, no gradients, no unnecessary gridlines
2. **Clear labels** — axis labels with units, legend with full method names
3. **Legend placement** — outside plot if >4 items: `ax.legend(bbox_to_anchor=(1.02, 1), loc='upper left')`
4. **Consistent colors** — "Ours" always uses `COLORS[0]` (blue), baselines use grey/orange
5. **Font readable at print size** — minimum 10pt after scaling in the paper
6. **Save as PDF** for vector graphics: `plt.savefig('paper/figures/NAME.pdf')`
7. **Figure size** — single column: `(6, 4)`, full width: `(10, 4.5)`

## Steps

1. Read `docs/experiment_results.md` for numerical data
2. Determine which plot type best represents the data
3. Write Python script using templates above
4. Run script, verify output PDF exists in `paper/figures/`
5. Check: labels readable, colors consistent, no clutter
