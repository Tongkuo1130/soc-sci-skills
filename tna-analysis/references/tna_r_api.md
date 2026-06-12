# TNA R Package API Quick Reference

> Based on tna CRAN package v1.2+. For full details, see `?tna` in R.

## Core Functions

### prepare_data(data, action, actor, time, order, time_threshold)
Prepares long-format event logs for TNA modeling.

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `data` | Yes | data.frame | Long-format event log |
| `action` | Yes | character | Column name for state/action codes |
| `actor` | Almost always | character | Column name for participant/group ID |
| `time` | Recommended | character | Column name for timestamps |
| `order` | Alternative | character | Column name for sequential ordering (if no timestamps) |
| `time_threshold` | Optional | numeric | Session split threshold in seconds (default 900 = 15 min) |

Returns: list with `$sequence`, `$meta`, `$long` components.

### tna(data, model = "tna")
Builds a transition network model.

| Parameter | Description |
|-----------|-------------|
| `data` | Output from `prepare_data()`, or wide-format data.frame, or matrix |
| `model` | `"tna"` (default), `"ftna"`, `"ctna"`, `"atna"`, `"wtna"` |

**Model type decision:**
- `"tna"` — Row-normalized transition probabilities (default; equal-length sequences)
- `"ftna"` — Raw frequency counts (unequal-length, descriptive focus)
- `"ctna"` — Co-occurrence (undirected; when order doesn't matter)
- `"atna"` — Attention-weighted (recent events weighted higher; long sequences)
- `"wtna"` — Weighted probability (corrects for sequence length imbalance)

Auto-detect recommendation: if `cv(sequence_lengths) > 0.5`, suggest ftna or wtna over tna.

Returns: list with `$weights`, `$inits`, `$labels`, `$data`.

### reliability(model, n = 1000)
Split-half reliability check. MUST run before any interpretation.

| Check | Threshold | Action if fail |
|-------|-----------|----------------|
| Pearson r | > 0.8 required, > 0.9 preferred | Diagnose: short sequences? too many states? small sample? |

### prune(model, method = "bootstrap", n = 1000, ...)
Filters non-significant edges.

| method | When to use |
|--------|-------------|
| `"bootstrap"` | Default. Data-driven, non-parametric. Needs sufficient samples. |
| `"disparity"` | Fast. Assumes uniform null model. Small sample fallback. |

Returns: pruned model object. Check `summary(pruned)` for retained edge ratio.

### plot(model, minimum, cut, layout, ...)
Visualizes TNA network.

Key parameters:
- `minimum` — hide edges below this probability (e.g., 0.05)
- `cut` — fade edges below this probability (e.g., 0.1)
- `layout` — `"circle"`, `"layout_with_fr"` (force-directed), etc.

### centralities(model)
Computes 9 centrality measures: InStrength, OutStrength, ClosenessIn, ClosenessOut, Closeness, Betweenness, BetweennessRSP, Diffusion, Clustering.

### stability(model, metric = "InStrength")
Case-dropping stability analysis. Standard: correlation > 0.7 after removing 50% of cases.

### communities(model, method = "spinglass")
Community detection. `method`: `"spinglass"` (recommended), `"walktrap"`, `"louvain"`.

### cliques(model, min_prob = 0.05)
Detects tightly-connected state subsets.

### clustering(prepared, method = "edit_distance", k = 3)
Sequence-based clustering of interaction patterns.

### permutation_test(model_a, model_b, n_permutations = 1000)
Permutation test comparing two TNA models. Returns p-values and effect sizes.

## R Package Installation

```r
install.packages("tna")  # CRAN
# OR
remotes::install_github("sonsoleslp/tna")  # Development
```

## Required Dependencies

igraph, TraMineR, Rcpp, ggplot2. Auto-installed by `install.packages("tna")`.

## Reading CSV Data in R (Chinese Encoding)

```r
# Try encoding detection first
d <- tryCatch(
  read.csv("data.csv", stringsAsFactors = FALSE),
  error = function(e) NULL
)
if (is.null(d)) {
  d <- read.csv("data.csv", fileEncoding = "UTF-8", stringsAsFactors = FALSE)
}
if (any(grepl("[\\x80-\\xff]", names(d)))) {
  d <- read.csv("data.csv", fileEncoding = "GBK", stringsAsFactors = FALSE)
}
# For semicolon-separated CSVs (Excel non-English locale)
d <- read.csv2("data.csv", stringsAsFactors = FALSE)
```

## Plot Output in Non-Interactive R

```r
png("output/network.png", width = 1200, height = 900, res = 150)
plot(pruned, minimum = 0.05, cut = 0.1)
dev.off()
```

## Detecting R Installation on Windows

```bash
# Try common locations
where Rscript 2>/dev/null
# or
ls "C:/Program Files/R/R-4."*/bin/Rscript.exe 2>/dev/null
# or check registry (PowerShell)
Get-ChildItem "HKLM:\SOFTWARE\R-core\R" -ErrorAction SilentlyContinue
```
