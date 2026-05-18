---
description: Create or revise a publication-quality data visualization following Tufte/Few/Doumont principles. Provide a description of the figure you want, optionally with a data source.
---

# Data Visualization Skill

Create or revise publication-quality figures for this research project. Every figure must follow these rules strictly.

## Core Philosophy

**Maximise the data-ink ratio** (Tufte). Every mark on the page must encode data or aid comprehension. Remove everything else. No decoration, no chartjunk, no gratuitous effects.

## Mandatory Rules

### Spines and Axes

- **Spines:** only bottom and left. Top and right spines must be removed (`sns.despine()` or `ax.spines["top"].set_visible(False)` etc.).
- **Grid lines:** do NOT add grid lines unless the user explicitly requests them or the chart type strictly requires them (e.g., a Cleveland dot plot where alignment to a shared axis is the whole point). Default is no grid.
- **Axis ticks:** use sparingly. Remove minor ticks. Reduce major tick count to the minimum needed for orientation. Use `MaxNLocator(nbins=...)` or explicit tick positions.
- **Y-axis labels:** NEVER rotate. Labels must be horizontal — `rotation=0`, `ha="right"`, and `ma="left"` (multi-line alignment) are all required. `ma="left"` left-aligns wrapped lines so a multi-line label reads as a tidy block instead of a ragged right edge. If a label is too long, break it into multiple lines and align properly:
  ```python
  ax.set_ylabel("Share of\nmonthly posts", rotation=0, ha="right", ma="left", y=1, labelpad=...)
  # or for tick labels:
  ax.set_yticklabels(
      [textwrap.fill(l, 20) for l in labels],
      rotation=0, ha="right", ma="left",
  )
  ```

### Labels vs. Legends

- **Prefer direct labels on data elements** over a separate legend. Place the label text next to or on the element it describes.
- **Label colour must match element colour.** If a bar is `#4477AA`, its label text must also be `#4477AA`.
- A legend is acceptable ONLY when direct labelling would cause clutter (e.g., overlapping time series with >5 lines in a single panel). Even then, prefer annotating a few key series directly and using a minimal legend for the rest.
- When a legend is unavoidable: `frameon=False`, outside the plot area, no border, no background fill.

### Bar Charts for Category Counts

- **Orientation:** always horizontal (`barh`), not vertical.
- **Sort order:** largest category at the top, smallest at the bottom. The reader's eye should scan top-down by magnitude.
- **Labels:** place the count/value label at the end of each bar (inside or outside depending on bar length), coloured to match the bar.
- No grid lines on the value axis. The direct labels make them redundant.

### Colour

- Use colour sparingly and with meaning. Default to a muted qualitative palette (Tol, ColorBrewer).
- Never encode meaning in colour alone — pair with shape, pattern, position, or direct label.
- For sequential data use a single-hue sequential palette. For diverging data, use a diverging palette centred on the meaningful midpoint.

### Information Layering (Doumont)

Every figure must communicate at three levels:
1. **Message (3-second read):** the figure title is a declarative sentence stating the finding. Example: "Forum activity drops 72% on Shabbat". NOT "Weekly activity patterns".
2. **Support:** the plot itself — the data that substantiates the title.
3. **Detail:** annotations, reference lines, precise values for close reading.

### Typography and Sizing

- Sans-serif font family (Helvetica, Source Sans, or system sans-serif).
- Base font size: 10pt for body text, axis labels. Title: 12-13pt, bold.
- `figure.dpi`: 300 for saved files.
- Save both PNG and SVG to `bhol_public_sphere/figures/`.
- Use `constrained_layout=True` or `bbox_inches="tight"` to prevent clipping.

### Matplotlib Setup Block

Start every figure script or function with:

```python
import matplotlib.pyplot as plt
import seaborn as sns

sns.set_theme(style="ticks", context="paper")
plt.rcParams.update({
    "font.family": "sans-serif",
    "font.size": 10,
    "axes.spines.top": False,
    "axes.spines.right": False,
    "axes.linewidth": 0.6,
    "xtick.major.width": 0.6,
    "ytick.major.width": 0.6,
    "figure.facecolor": "white",
    "axes.facecolor": "white",
    "axes.grid": False,
    "savefig.dpi": 300,
    "savefig.facecolor": "white",
})
```

Then after creating axes, always call `sns.despine(ax=ax)`.

## Output

- Save to `bhol_public_sphere/figures/` as both `.png` (dpi=300) and `.svg`.
- Use descriptive filenames: `shabbat_weekly_pattern.png`, not `fig1.png`.

## Anti-Patterns (never do these)

- Vertical bar charts for category comparisons
- Rotated y-axis labels (always `rotation=0`, `ha="right"`, `ma="left"`)
- Grid lines by default
- Legends when direct labels would work
- Labels in a different colour than the element they describe
- Descriptive titles ("Bar chart of topic counts") instead of declarative findings
- Dual-axis charts
- 3D effects, drop shadows, gradient fills
- Excessive tick marks
- Right or top spines
