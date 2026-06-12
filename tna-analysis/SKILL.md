---
name: tna-analysis
description: "Transition Network Analysis interactive analysis wizard. Use R package tna for temporal state sequence analysis. Triggers when user asks about TNA, transition networks, behavior sequences, sequential interaction data, temporal process modeling, or any Chinese keyword like 转换网络分析, 行为序列分析, 交互模式分析. Triggers on text mention alone -- file upload is a plus, not a requirement. Full pipeline from environment setup through interpretation with statistical validation."
agent_created: true
---

# TNA Analysis Skill

## Overview

This skill provides a guided, statistically-validated Transition Network Analysis workflow.
TNA models temporal processes as directed weighted networks where nodes are discrete states
and edges represent transition probabilities. The skill enforces a mandatory four-layer
validation framework and produces fully reproducible analysis outputs.

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
    → [Model build → Reliability gate → Prune gate → Centrality stability gate]
    → [Optional: communities, cliques, clustering, group compare]
    → [Output package: scripts, plots, CSVs, report, sessionInfo]
```

## Phase 0: Intent Clarification

Before any analysis, ask the user what they want from TNA. Present as a simple choice:

1. **Overall transition patterns** — "What does the whole interaction process look like?"
2. **Hub state identification** — "Which state is most central / acts as a bottleneck?"
3. **Group comparison** — "Do Group A and Group B have different interaction patterns?"
4. **Behavioral clustering** — "Are there distinct types of interaction sequences?"
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

If the package is missing: instruct the user to run `install.packages("tna")` in R. Do NOT run this command automatically—CRAN mirror selection is interactive.

If `tna` installs but dependencies fail (common on Windows without Rtools), suggest installing the binary version: `install.packages("tna", type = "binary")`.

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
if (is.null(d) || any(grepl('[\\x{4e00}-\\x{9fff}]', names(d), perl = FALSE)))) {
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

Auto-calculate the CV (coefficient of variation) of sequence lengths:

```r
seq_lengths <- sapply(prepared$sequence, function(x) sum(!is.na(x)))
cv <- sd(seq_lengths) / mean(seq_lengths)
```

Present recommendation:

- CV < 0.5 → Standard TNA (default)
- CV > 0.5 → Recommend FTNA or WTNA (sequences vary too much in length for probability normalization)

Explain the choice in plain terms and confirm with user.

### Step 3.3: Bootstrap Iterations

Default: n = 1000. Explain: "Bootstrap 迭代次数越多，边的显著性检验越精确，但计算时间越长。1000 次对大多数数据够用。"

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
model <- tna(prepared, model = "CONFIRMED_MODEL_TYPE")
```

Report: number of nodes, number of edges (including zero-probability), initial probability distribution.

### Step 4.3: GATE 1 — Reliability Check (MANDATORY, NO EXCEPTIONS)

```r
rel <- reliability(model, n = 1000)
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
pruned <- prune(model, method = "bootstrap", n = 1000)
```

Report:
- Total edges before pruning
- Edges retained after pruning
- Retention ratio

If retention < 10%, warn: "数据中的信号很弱，网络接近随机。幸存的几条边不应过度解读。" Still show results, but with strong caveat.

### Step 4.5: Generate Network Plot

```r
png("output/tna_network.png", width = 1200, height = 900, res = 150)
plot(pruned, minimum = 0.05, cut = 0.1)
dev.off()
```

Save as file, display to user. Explain: nodes = states, arrows = transitions, thicker = higher probability, colored borders = initial probability.

## Phase 5: Analysis Execution

Execute only the analyses corresponding to the user's intent from Phase 0.

### 5.1: Centrality Analysis

```r
cent <- centralities(pruned)
```

Present as a ranked table. Highlight top InStrength (most popular target), top OutStrength (most frequent source), top Betweenness (bridge node).

### 5.2: GATE 3 — Centrality Stability (MANDATORY if interpreting centralities)

```r
stab <- stability(pruned, metric = "InStrength")
```

Standard: correlation must remain > 0.7 after dropping 50% of cases. Present result and stability assessment for each centrality metric used.

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

```r
clust <- clustering(prepared, method = "edit_distance", k = 3)
```

Present cluster sizes, characteristic patterns per cluster, and covariate associations (from metadata).

### 5.6: GATE 4 — Group Comparison with Permutation Test (if requested)

```r
# Build per-group models
model_a <- tna(subset(prepared, CONDITION_COL == "GROUP_A"), model = "CONFIRMED_TYPE")
model_b <- tna(subset(prepared, CONDITION_COL == "GROUP_B"), model = "CONFIRMED_TYPE")
pruned_a <- prune(model_a, method = "bootstrap", n = 1000)
pruned_b <- prune(model_b, method = "bootstrap", n = 1000)
perm_test <- permutation_test(pruned_a, pruned_b, n_permutations = 1000)
```

Report:
- Overall network difference (p-value, effect size)
- Significantly different edges (with p-values and direction)
- Significantly different centrality values

## Phase 6: Output Packaging

After analysis is complete, package ALL deliverables into an output directory:

```
output/
├── tna_analysis.R           # Full reproducible script
├── transition_matrix.csv    # Row-normalized probability matrix
├── centrality_table.csv     # All centrality measures for all nodes
├── tna_network.png          # Network plot (pruned)
├── reliability.png          # Split-half reliability plot
├── stability.png            # Case-dropping stability plot
├── communities.png          # Community detection plot (if run)
├── sequences.png            # Sequence index plot
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
- 模型类型: TNA (标准)
- 状态数: 7
- 序列数: 48
- 会话数: 112
- 可靠性: Pearson r = 0.94
- Bootstrap 剪枝: 25/42 边保留 (59.5%)
- InStrength 稳定性: r = 0.82 (移除50%数据后)
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
- [ ] Centrality stability checked (r > 0.7 at 50% data removal)
- [ ] Group comparison used permutation test (not visual comparison)
- [ ] All plots saved as files (not just displayed)
- [ ] Reproducible .R script saved with actual parameter values
- [ ] sessionInfo() captured
- [ ] Findings written in plain language with statistical evidence
- [ ] BibTeX citation included in report: Saqr et al. (2025), LAK '25, DOI: 10.1145/3706468.3706513
