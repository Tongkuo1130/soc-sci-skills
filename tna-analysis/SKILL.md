---
name: tna-analysis
description: "Transition Network Analysis interactive analysis wizard. Use R package tna for temporal state sequence analysis. Triggers when user asks about TNA, transition networks, behavior sequences, sequential interaction data, temporal process modeling, or any Chinese keyword like 转换网络分析, 行为序列分析, 交互模式分析. Triggers on text mention alone -- file upload is a plus, not a requirement. Full pipeline from environment setup through interpretation with statistical validation."
agent_created: true
---

# TNA Analysis Skill

## Overview

This skill provides a guided, statistically-validated Transition Network Analysis workflow.
TNA models temporal processes as directed weighted networks where nodes are discrete states
and edges represent transition probabilities. The skill enforces mandatory statistical validation
gates (reliability → pruning → group comparison) and produces fully reproducible analysis outputs.

## When to Use

Invoke this skill when the user's intent involves any of:

- Analyzing sequential behavior data (event logs, coded interactions, clickstreams)
- Understanding which states transition to which, with what probability
- Identifying hub states and transition bottlenecks
- Comparing transition network structures between groups
- Discovering behavioral clusters or tactics from sequences
- Keywords: TNA, transition network, 转换网络, 行为序列, process mining, 交互模式, 时序分析, sequential analysis, state transitions, Markov model

## CRITICAL: Core Principle

**Every analysis conclusion MUST be backed by actual `Rscript` output. NEVER simulate or fabricate TNA results by reading data directly and guessing probabilities.** If R is unavailable, the skill's boundary stops at "generate an executable R script"—it must not produce findings beyond that line.

## Workflow Overview

```
[User intent] → [Environment check] → [Data load & diagnose] → [Parameter negotiate]
    → [Model build → Reliability gate → Prune gate]
    → [Optional: centrality, communities, cliques, group compare]
    → [Output package: scripts, plots, CSVs, report, sessionInfo]

⚠️ **tna v1.2.3 note**: `stability()` and `clustering()` are not available in the current CRAN release.
For centrality interpretation, note this limitation; for clustering, suggest external packages.
```

## Phase 0: Intent Clarification

Before any analysis, ask the user what they want from TNA. Present as a simple choice:

1. **Overall transition patterns** — "What does the whole interaction process look like?"
2. **Hub state identification** — "Which state is most central / acts as a bottleneck?"
3. **Group comparison** — "Do Group A and Group B have different interaction patterns?"
4. **Behavioral clustering** — "Are there distinct types of interaction sequences?" ⚠️ v1.2.3 的 `clustering()` 不可用，需外部包。
5. **All of the above** — Full pipeline

Skip this only if the user's initial message already specifies what they want.

## Phase 1: Environment Detection & Setup

### Step 1.1: Detect R Installation

Run a command to check if R is installed. On Windows, try multiple detection strategies:

```bash
where Rscript 2>/dev/null || ls "C:/Program Files/R/R-4."*/bin/Rscript.exe 2>/dev/null
```

If R is not found, present the user with options:

> 「检测到你的电脑未安装 R。TNA 分析需要 R 环境。我可以帮你——」
> - A. **自动安装 R**（下载约 80MB，需 2-3 分钟）
> - B. **生成完整分析脚本**——你拿去有 R 的机器跑，结果贴回来我帮你解读
> - C. **用 TNA Shiny App**（浏览器在线分析，无需安装）：sonsoleslp.shinyapps.io/tna-app/

If the user chooses A, guide them through installing R (do NOT silently auto-install system software). After installation, verify it's in PATH.

### Step 1.2: Check tna Package

```bash
Rscript -e 'library(tna)' 2>&1
```

If the package is missing: install it. On Windows, use a personal library path to avoid permission errors:

```r
install.packages("tna", repos = "https://cloud.r-project.org")
```

Or if the system library is not writable:
```r
my_lib <- "C:/Users/USERNAME/Documents/R/win-library/4.6"
install.packages("tna", repos = "https://cloud.r-project.org", lib = my_lib)
.libPaths(c(my_lib, .libPaths()))
```

✅ Using `repos = "https://cloud.r-project.org"` avoids the interactive CRAN mirror prompt.

### Step 1.3: Record Environment Info

Once R + tna are confirmed, capture:

```bash
Rscript -e 'cat(R.version$version.string); cat("\n"); cat(as.character(packageVersion("tna")))'
```

Store this for the final `sessionInfo()` output and reproducibility trace.

## Phase 2: Data Loading & Diagnosis

### Step 2.1: Detect File Encoding and Delimiter

Chinese CSV files are encoding minefields (GBK, UTF-8, UTF-8-BOM). Read the file with `file` command or Python to detect encoding before loading into R:

```bash
# Quick encoding check (Python)
python -c "
with open('FILE_PATH', 'rb') as f:
    raw = f.read(200)
print('BOM:', raw[:3] if raw[:3] in [b'\xef\xbb\xbf', b'\xff\xfe', b'\xfe\xff'] else 'none')
print('Preview:', raw[:200])
"
```

Try loading in R with fallback encodings:

```r
d <- tryCatch(read.csv("FILE_PATH", stringsAsFactors = FALSE), error = function(e) NULL)
if (is.null(d) || any(grepl('[\\x{4e00}-\\x{9fff}]', names(d), perl = FALSE))) {
  d <- tryCatch(read.csv("FILE_PATH", fileEncoding = "UTF-8", stringsAsFactors = FALSE),
                error = function(e) read.csv("FILE_PATH", fileEncoding = "GBK", stringsAsFactors = FALSE))
}
```

Also try `read.csv2()` for semicolon-separated CSVs (Excel non-English locale).

### Step 2.2: Auto-Detect Data Structure

After successful load, analyze the data and report:

- **Number of rows and columns**
- **Column names** (if Chinese, translate to English equivalents and confirm with user)
- **Column types** (character, numeric, datetime)
- **For each character column**: unique value count (identify likely `action` column: 3-15 unique values)
- **For each ID-like column**: repeat counts (identify likely `actor` column)
- **For each time-like column**: format detection results
- **Missing values**: count and percentage per column
- **Extra columns**: list all columns beyond action/actor/time → these auto-become metadata

### Step 2.3: Confirm Column Mapping

Present detected columns for user confirmation:

> 「我猜测你的数据列对应关系如下——」
> - **行为/状态列（action）**: "Behavior" → 共 7 种状态：监控、执行、反思、求助、计划、调节、中断
> - **参与者列（actor）**: "StudentID" → 48 个独立参与者
> - **时间列（time）**: "Timestamp" → 时间范围 2024-03-01 至 2024-03-15
> - **分组列**: "Condition" → high / low
>
> 「是否正确？如有误请告诉我正确的列名。」

### Step 2.4: Data Quality Diagnostics

Run and report:

1. **State count warning**: If states > 15, warn that the network will be hard to interpret. Suggest merging similar states.
2. **Sequence length distribution**: Calculate min, median, mean, max events per actor. If mean < 5, warn that Markov probability estimates will be unstable.
3. **Actor event imbalance**: Flag actors with disproportionately few events.
4. **Empty/missing state values**: Report and offer to remove.

### Step 2.5: Data Cleaning

Distinguish between **technical cleaning** (auto-apply after confirmation) and **analytical decisions** (must be user's call):

| Technical (auto-offer) | Analytical (must ask) |
|------------------------|----------------------|
| Remove rows with empty action values | Merge rare state categories |
| Fix time format inconsistencies | Exclude outlier actors |
| Remove duplicate timestamps | Change coding granularity |

Always report what was cleaned and how many rows were affected.

## Phase 3: Parameter Negotiation

### Step 3.1: time_threshold

Explain what this does in plain language, then suggest a value based on context:

> 「`time_threshold` 控制"多久没活动算一次会话中断"。默认 15 分钟（900 秒）。」
> - 如果是课堂讨论：10-20 分钟合理
> - 如果是在线学习日志：可能用几小时甚至几天
> - 你的场景下，建议设为多少秒？」

Add: 「关键结论建议做敏感性分析——换一个阈值重新跑，看结论是否稳健。」

### Step 3.2: Model Type

In tna v1.2.3, `tna()` auto-dispatches — no explicit `model` parameter needed. The function uses row-normalized transition probabilities by default.

If the user needs a different model (frequency, co-occurrence, etc.), use `build_model()` directly:
```r
model <- build_model(prepared, type = "relative")  # default (same as tna())
# Alternatives: "frequency", "co-occurrence"
```

## Phase 4: Model Building with Mandatory Validation Gates

### Step 4.1: prepare_data()

```r
prepared <- prepare_data(
  d,
  action = "CONFIRMED_ACTION_COL",
  actor  = "CONFIRMED_ACTOR_COL",
  time   = "CONFIRMED_TIME_COL",
  time_threshold = CONFIRMED_THRESHOLD
)
```

Report output statistics:
- Number of sequences created
- Number of sessions split
- Unique states and their frequencies
- Mean/median events per sequence

### Step 4.2: Build Model

```r
model <- tna(prepared)    # v1.2.3: no model parameter
```

Report: number of nodes, number of edges, transition probability matrix.

### Step 4.3: GATE 1 — Reliability Check (MANDATORY, NO EXCEPTIONS)

```r
rel <- reliability(model, n = 1000)
r_pearson <- rel$summary$mean[rel$summary$metric == "Pearson"]  # v1.2.3 output structure
```

Present result clearly:

| Pearson r | Verdict | Action |
|-----------|---------|--------|
| > 0.9 | PASS | Proceed |
| 0.8 — 0.9 | BORDERLINE | Proceed with caution note |
| < 0.8 | FAIL | HALT. Diagnose: short sequences? too many states? small sample? random data? |

If FAIL, do NOT continue to downstream analysis. Present diagnostic recommendations.

### Step 4.4: GATE 2 — Bootstrap Pruning (MANDATORY)

```r
pruned <- prune(model)    # v1.2.3: no method/n params; auto-selects pruning method
```

Report:
- Total edges before pruning
- Edges retained after pruning
- Retention ratio

If retention < 10%, warn: "数据中的信号很弱，网络接近随机。幸存的几条边不应过度解读。" Still show results, but with strong caveat.

### Step 4.5: Generate Network Plot

```r
png("output/tna_network.png", width = 1200, height = 900, res = 150)
plot(pruned, minimum = 0.05, edge_cutoff = 0.1)   # v1.2.3: 'cut' deprecated, use 'edge_cutoff'
dev.off()
```

Save as file, display to user. Explain: nodes = states, arrows = transitions, thicker = higher probability, colored borders = initial probability.

## Phase 5: Analysis Execution

Execute only the analyses corresponding to the user's intent from Phase 0.

### 5.1: Centrality Analysis

```r
cent <- centralities(pruned)    # v1.2.3: returns data.frame with 'state' column
cent <- cent[order(-cent$InStrength), ]  # sort by InStrength
```

Present as a ranked table using `cent$state`, `cent$InStrength`, `cent$OutStrength`, `cent$Betweenness`, `cent$Closeness`. Highlight top InStrength (most popular target), top OutStrength (most frequent source), top Betweenness (bridge node).

### 5.2: Centrality Stability

⚠️ **`stability()` is not available in tna v1.2.3.** Include this caveat in the report: "中心性指标未经过案例剔除稳定性检验（tna v1.2.3 中 stability() 函数不可用）。"

### 5.3: Community Detection (if requested)

```r
comms <- communities(pruned, method = "spinglass")
plot(pruned, communities = comms)
```

Present communities as named groups, explain what each cluster represents.

### 5.4: Clique Detection (if requested)

```r
cls <- cliques(pruned, min_prob = 0.05)
```

Report dyads (bidirectional pairs) and triads (fully connected triplets).

### 5.5: Sequence Clustering (if requested)

⚠️ **`clustering()` is not available in tna v1.2.3.** Suggest external alternatives:
- **TraMineR + cluster**: `seqdist()` for distance matrix → `hclust()` or `pam()`
- **R package `ClusterR`**: k-means on sequence-derived features

### 5.6: Group Comparison with Permutation Test (if requested)

**Option A — `group_tna()` (v1.2.3, recommended):**

```r
gt <- group_tna(d, action = "Action", actor = "Actor", time = "Time",
                group = "CONDITION_COL", time_threshold = CONFIRMED_THRESHOLD)
p_a <- prune(gt$GROUP_A)
p_b <- prune(gt$GROUP_B)
```

Plot side-by-side for descriptive comparison:
```r
png("output/group_comparison.png", width = 1800, height = 800, res = 150)
par(mfrow = c(1, 2))
plot(p_a, minimum = 0.05, edge_cutoff = 0.1, main = "Group A")
plot(p_b, minimum = 0.05, edge_cutoff = 0.1, main = "Group B")
par(mfrow = c(1, 1))
dev.off()
```

**Option B — `permutation_test()` (statistical comparison):**

```r
perm <- permutation_test(p_a, p_b, n_permutations = 1000)
```

⚠️ **Memory warning**: 80+ sequences with 1000 permutations may need >10GB. For large datasets, reduce `n_permutations` to 100.

## Phase 6: Output Packaging

After analysis is complete, package ALL deliverables into an output directory:

```
output/
├── tna_analysis.R           # Full reproducible script
├── transition_matrix.csv    # Row-normalized probability matrix
├── centrality_table.csv     # All centrality measures for all nodes
├── tna_network.png          # Network plot (pruned)
├── group_comparison.png     # Side-by-side group networks (if run)
├── communities.png          # Community detection plot (if run)
├── analysis_report.md       # Human-readable findings report
└── session_info.txt         # R sessionInfo() output
```

### The Reproducible Script

The `tna_analysis.R` file MUST contain ALL actual executed code with concrete parameter values (not placeholders). A researcher should be able to `source("tna_analysis.R")` and reproduce every number.

### The Analysis Report

Structure the report as plain-language findings, not R output dumps:

```
## 核心发现

1. [Finding 1 with plain explanation]
2. [Finding 2 with plain explanation]
...

## 模型信息
- 模型类型: TNA (行归一化概率)
- 状态数: 7
- 序列数: 48
- 会话数: 112
- 可靠性: Pearson r = 0.94
- Bootstrap 剪枝: 25/42 边保留 (59.5%)
- ⚠️ 中心性稳定性: 未检验（tna v1.2.3 不提供 stability()）
- 组间比较: p = 0.003（置换检验，1000次）
```

### sessionInfo()

Always output `sessionInfo()` at the end of the reproducible script and the report.

## Fallback Mode: Script Generation Only

When R is unavailable and user chooses script generation:

1. Read and analyze the user's data file (detect columns, types, unique values)
2. Generate a customized `.R` script with column names already mapped
3. Provide clear instructions: "Copy this script and your data file to a machine with R, then run `source('tna_analysis.R')` in RStudio."
4. Include comments in the script explaining each step in Chinese
5. Include instructions for how to bring results back for interpretation
6. Do NOT produce any "preliminary findings" or simulated results

## Error Handling

| Error | Response |
|-------|----------|
| R crashes mid-analysis | Report: which step failed, what intermediate results exist, how to resume |
| Bootstrap removes nearly all edges | Warn but continue; interpret remaining edges with strong caveats |
| Permutation test not significant | Report as valid finding ("no detectable difference"); check within-group variability |
| R memory exhausted | Suggest sampling or aggregating; check object sizes |
| Time format unparseable | Stop and ask user to specify format; do not assume |
| State column has numeric values | Confirm they should be treated as categories; warn if >15 unique values |
| Actor column missing | Warn: all events will be treated as one sequence (usually incorrect) |

## Language & Communication Rules

1. Interact in Chinese unless the user explicitly prefers English
2. Use plain language for all explanations—assume the user has zero TNA knowledge
3. Every analysis step output must include: what was done → why → result → what it means
4. For parameter decisions: explain the trade-off, give a recommendation with rationale, but let the user decide
5. Never say "the results show that X leads to Y"—say "students tended to transition from X to Y (p = 0.XX)"
6. For negative findings (no significant difference, low reliability), normalize them: "没有显著差异也是发现"

## Quick Checklist

Before delivering final results, verify:

- [ ] Reliability gate passed (r > 0.8)
- [ ] Edges pruned via bootstrap (report retention ratio)
- [ ] Group comparison used permutation test (not visual comparison); if OOM, use descriptive side-by-side with caveat
- [ ] All plots saved as files (not just displayed)
- [ ] Reproducible .R script saved with actual parameter values
- [ ] `sessionInfo()` captured (include R + tna versions)
- [ ] Findings written in plain language with statistical evidence
- [ ] ⚠️ tna v1.2.3 limitations noted: no `stability()`, no `clustering()`; `edge_cutoff` replaces `cut` in plot()
- [ ] BibTeX citation included in report: Tikka, S., Lopez-Pernas, S., & Saqr, M. (2025). tna: An R Package for Transition Network Analysis. Applied Psychological Measurement. DOI: 10.1177/01466216251348840
