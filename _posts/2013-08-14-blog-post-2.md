---
title: 'Probability Distribution graph for static modeling'
date: 2025-11-14
permalink: /posts/2013/08/blog-post-2/
tags:
  - Static modeling
---

Static modeling is the process of building a 3D reservoir model. In contrast, probabilistic static modeling takes uncertainty into account and creates many realizations of the model to generate distributions of key properties, such as GIIP, OOIP, net-to-gross, porosity, and permeability, which are usually reported as P10, P50, and P90 values.

When we calculate volumetrics using a probabilistic approach, we obtain many possible volume values. By plotting these results on a probability distribution graph, we can identify representative values for each probability level, such as P10, P50, and P90.

<img width="974" height="605" alt="image" src="https://github.com/user-attachments/assets/2ff3db10-06b7-4337-83ca-64709a9a7044" />

---python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# 1) Read GIIP File
# If it is a .csv file, use read_csv instead
df = pd.read_csv("GIIP.csv")

# Combine values from all desired columns
cols = ["Short Range", "Medium Range", "Long Range"]
giip = df[cols].values.flatten()      # Convert to 1D array
giip = giip[~np.isnan(giip)]         # Remove NaN values if they exist

# 2) Prepare data for plotting
giip_sorted = np.sort(giip)          # Sort from low to high
n = len(giip_sorted)

# Use empirical CDF and reverse it to get "probability of exceedance"
ranks = np.arange(1, n + 1)
p_cdf = ranks / (n + 1)              # 0–1 (Bottom up)
p_exceed = 1 - p_cdf                 # 1–0 (Like the example graph)

# 3) Find P90, P50, P10
#    (Using general reserve volume survey definition)
#    P90 = 10th percentile
#    P50 = 50th percentile
#    P10 = 90th percentile
P90_val = np.quantile(giip_sorted, 0.10)
P50_val = np.quantile(giip_sorted, 0.50)
P10_val = np.quantile(giip_sorted, 0.90)

print(f"P90 GIIP ≈ {P90_val:.1f} BSCF")
print(f"P50 GIIP ≈ {P50_val:.1f} BSCF")
print(f"P10 GIIP ≈ {P10_val:.1f} BSCF")

# 4) Plot the graph as in the figure

fig, ax = plt.subplots(figsize=(8, 5))

# GIIP vs probability points
ax.scatter(giip_sorted, p_exceed, s=20)

# Horizontal lines for different P levels
ax.axhline(0.90, color="blue",  linestyle="--", linewidth=1)
ax.axhline(0.50, color="red",   linestyle="--", linewidth=1)
ax.axhline(0.10, color="purple",linestyle="--", linewidth=1)

# Vertical lines for representative GIIP P values
ax.axvline(P90_val, color="blue",  linestyle="--", linewidth=1)
ax.axvline(P50_val, color="red",   linestyle="--", linewidth=1)
ax.axvline(P10_val, color="purple",linestyle="--", linewidth=1)

# Arrows (vertical) pointing down to the x-axis
ax.annotate("", xy=(P90_val, 0.0),  xytext=(P90_val, 0.90),
            arrowprops=dict(arrowstyle="->", color="blue"))
ax.annotate("", xy=(P50_val, 0.0),  xytext=(P50_val, 0.50),
            arrowprops=dict(arrowstyle="->", color="red"))
ax.annotate("", xy=(P10_val, 0.0),  xytext=(P10_val, 0.10),
            arrowprops=dict(arrowstyle="->", color="purple"))

# Add labels
ax.set_title("A-01", fontsize=16)
ax.set_xlabel("GIIP (BSCF)")
ax.set_ylabel("Probability of Exceedance")

# Legend similar to the figure
from matplotlib.lines import Line2D
legend_lines = [
    Line2D([0], [0], color="blue",  lw=2),
    Line2D([0], [0], color="red",   lw=2),
    Line2D([0], [0], color="purple",lw=2),
]
ax.legend(legend_lines, ["P90", "P50", "P10"], loc="lower right")

ax.grid(True)
plt.tight_layout()
plt.show()
------
