# TNA R Package API Quick Reference

> Verified against tna CRAN package v1.2.3. For full details, see `?tna` in R.

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

**Returns (v1.2.3)**: `tna_data` list with:
- `$statistics` — list: `total_sessions`, `total_actions`, `max_sequence_length`, `unique_users`, `time_range`
- `$sequence_data` — tibble: wide-format sequences (rows = sessions, cols = positions)
- `$meta_data` — tibble: metadata per session
- `$long_data` — tibble: original data with parsed time
- `$time_data` — tibble: parsed timestamps

**Sequence length extraction**:
```r
seq_lengths <- prepared$sequence_data |> apply(1, function(r) sum(!is.na(r)))
```

### tna(x, ...)

Builds a transition network model. In v1.2.3, the function dispatches automatically — no explicit `model` parameter.

```r
model <- tna(prepared)        # Correct for v1.2.3
# ❌ model <- tna(prepared, model = "tna")  # Error in v1.2.3
```

**Returns**: `tna` object with `$weights` (probability matrix), `$inits` (initial probabilities), `$labels` (state names), `$data`.

### reliability(model, n = 1000)

Split-half reliability check. MUST run before any interpretation.

**v1.2.3 output**: `tna_reliability` object with `$summary` tibble:

| metric | mean | sd | median | min | max |
|--------|------|----|--------|-----|-----|
| Mean Abs. Diff. | ... | ... | ... | ... | ... |
| Pearson | ... | ... | ... | ... | ... |

```r
rel <- reliability(model, n = 1000)
r_pearson <- rel$summary$mean[rel$summary$metric == "Pearson"]
```

| Pearson r | Threshold | Action if fail |
|-----------|-----------|----------------|
| > 0.9 | PASS | Proceed |
| > 0.8 | BORDERLINE | Proceed with caution |
| < 0.8 | FAIL | Diagnose: short sequences? too many states? small sample? |

### prune(x, ...)

Filters non-significant edges. In v1.2.3, the default method auto-selects between bootstrap and disparity.

```r
pruned <- prune(model)        # Correct for v1.2.3
# ❌ prune(model, method = "bootstrap", n = 1000)  # Error in v1.2.3
```

**Returns**: pruned `tna` model. Check retained edges:
```r
n_total <- prod(dim(pruned$weights)) - nrow(pruned$weights)
n_retained <- sum(pruned$weights > 0) - nrow(pruned$weights)
```

### plot(model, minimum, edge_cutoff, layout, ...)

Visualizes TNA network.

**v1.2.3 key params**:
- `minimum` — hide edges below this probability (e.g., 0.05)
- `edge_cutoff` — fade edges below this probability (e.g., 0.1). **`cut` is deprecated in v1.2.3.**
- `layout` — `"circle"`, `"layout_with_fr"` (force-directed), etc.

```r
png("network.png", width = 1200, height = 900, res = 150)
plot(pruned, minimum = 0.05, edge_cutoff = 0.1)
dev.off()
```

### centralities(model)

Computes centrality measures. **v1.2.3 returns a data.frame** with columns:

| Column | Description |
|--------|-------------|
| `state` | State label (factor) |
| `InStrength` | Weighted indegree (hub target) |
| `OutStrength` | Weighted outdegree (hub source) |
| `ClosenessIn` / `ClosenessOut` / `Closeness` | Closeness centrality |
| `Betweenness` / `BetweennessRSP` | Betweenness centrality |
| `Diffusion` | Diffusion centrality |
| `Clustering` | Clustering coefficient |

```r
cent <- centralities(pruned)
cent <- cent[order(-cent$InStrength), ]  # Rank by InStrength
```

### communities(model, method = "spinglass")

Community detection. `method`: `"spinglass"` (recommended), `"walktrap"`, `"louvain"`.

⚠️ **v1.2.3 caveat**: spinglass may return `NULL` (non-convergence) with small/sparse networks.

### cliques(model, min_prob = 0.05)

Detects tightly-connected state subsets.

### group_tna(data, action, actor, time, group, time_threshold)

**v1.2.3 preferred approach for group comparisons.** Builds per-group models simultaneously.

```r
gt <- group_tna(d, action = "Action", actor = "Actor", time = "Time",
                group = "Performance", time_threshold = 86400)
p_high <- prune(gt$high)
p_low  <- prune(gt$low)
```

### permutation_test(x, y, n_permutations = 1000)

Permutation test comparing two pruned TNA models.

⚠️ **v1.2.3 caveat**: Memory-intensive. 80+ sequences with 1000 permutations can require >10GB RAM. Use `n_permutations = 100` for large datasets, or rely on `group_tna()` side-by-side descriptive comparison.

### NOT AVAILABLE in v1.2.3

| Function | Status | Alternative |
|----------|--------|-------------|
| `stability()` | Removed | Report centrality with caveat that stability check unavailable |
| `clustering()` | Removed | Use external sequence clustering (TraMineR + cluster) |
| `model` param in `tna()` | Removed | Auto-dispatches; use `build_model()` for manual control |

## R Package Installation

```r
install.packages("tna")  # CRAN
```

**Windows note**: Default library `C:/Program Files/R/...` is not writable. Use a personal library:

```r
my_lib <- "C:/Users/USERNAME/Documents/R/win-library/4.6"
install.packages("tna", repos = "https://cloud.r-project.org", lib = my_lib)
.libPaths(c(my_lib, .libPaths()))
```

⚠️ `install.packages("tna")` auto-installs 30+ dependencies (igraph, ggplot2, dplyr, etc.). With binary packages on Windows this is fast; from source requires Rtools.

## Reading CSV Data in R (Chinese Encoding)

```r
d <- tryCatch(
  read.csv("data.csv", stringsAsFactors = FALSE, fileEncoding = "UTF-8"),
  error = function(e) read.csv("data.csv", fileEncoding = "GBK", stringsAsFactors = FALSE)
)
```

## Plot Output in Non-Interactive R

```r
png("output/network.png", width = 1200, height = 900, res = 150)
plot(pruned, minimum = 0.05, edge_cutoff = 0.1)
dev.off()
```

## Detecting R Installation on Windows

```bash
where Rscript 2>/dev/null
# or
ls "C:/Program Files/R/R-4."*/bin/Rscript.exe 2>/dev/null
# or PowerShell
Get-ChildItem "HKLM:\SOFTWARE\R-core\R" -ErrorAction SilentlyContinue
```
