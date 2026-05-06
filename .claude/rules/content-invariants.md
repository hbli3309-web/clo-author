# Content Invariants

These are non-negotiable. Every agent checks against them. Violations are deductions, not suggestions. Critics cite invariant numbers (e.g., "violates INV-3") in their reports.

---

## Paper

**INV-1.** Every table has notes explaining key variables, sample, and data source — via `threeparttable` + `tablenotes` (traditional) or `talltblr` with `note{}` keys (tabularray).

**INV-2.** Every figure has a `\caption{}` with a note explaining what is shown, how to read it, and the data source.

**INV-3.** No `\hline` — use `\toprule`, `\midrule`, `\bottomrule` (booktabs). No vertical rules.

**INV-4.** Significance stars follow the journal profile. AEA journals: no stars, report standard errors and use confidence intervals. Default: stars with note defining thresholds.

**INV-5.** Abstract is 150 words or fewer.

**INV-6.** JEL codes and keywords present after the abstract.

**INV-7.** Notation is consistent across all sections — the same symbol means the same thing everywhere. Different concepts get different symbols.

**INV-8.** Every causal claim has a corresponding identification section. No causal language in descriptive papers.

**INV-9.** `biblatex` + `biber`, not `natbib` + `bibtex`.

**INV-10.** `hyperref` loaded second-to-last in preamble; `cleveref` loaded immediately after it.

**INV-11.** Numbers in text match the tables and figures exactly. No rounding discrepancies, no stale values.

**INV-12.** No titles inside ggplot/matplotlib figures. Titles go in LaTeX `\caption{}`. Panel labels ("Panel A: ...") inside multi-panel figures are fine.

**INV-13.** R/Python/Julia scripts export bare `tabular` environments — no `\begin{table}`, `\caption{}`, or notes. The paper's `main.tex` wraps them.

## Code

**INV-14.** `set.seed()` (or language equivalent) called exactly once, at the top of the main script, if any stochastic element exists.

**INV-15.** All packages/libraries loaded at the top of the script, before any data loading or computation.

**INV-16.** No absolute paths. All paths relative to project root via `here()` (R), `pathlib.Path` (Python), `joinpath(@__DIR__, ...)` (Julia), or project-root macros set in a central `_setup.do` (Stata) — typically `$root`, `$rawdata`, `$workingdata`, `$tempdata`, `$figure`, `$table` (lowercase paths; reserve `UPPER_SNAKE_CASE` for tunable constants like `$SEED`, `$N_BOOT`). No `cd` outside `_setup.do`.

**INV-17.** No growing vectors/lists in loops. Pre-allocate result containers or use vectorized operations.

**INV-18.** Output files go to the path specified by the Output Organization setting in `CLAUDE.md`.

**INV-19.** No prohibited functions: `setwd()` / `os.chdir()` / `cd()`, `rm(list = ls())`, `install.packages()` in scripts, `attach()` / `detach()`.

**INV-22.** Spec parameters are declared once as named macros / constants near the top of the script (or in `_setup.do` when shared across scripts) and referenced everywhere downstream. This covers — at minimum — sample-selection filters, time windows, dependent variable(s), treatment definition, control set, fixed-effects set, cluster level, and bandwidth. No hardcoded year ranges, sample filters, or variable names buried inline in `keep if`, `reghdfe`, or `csdid` calls. Stata: `global` for project-wide spec, `local` for script-scoped spec; UPPER_SNAKE_CASE for tunable constants. R/Python/Julia: a CONFIG block (or `_setup` module) of named constants at the top.

**INV-23.** Data processing and analysis are separated. Analysis scripts read cleaned data from disk and never overwrite the canonical cleaned files. If an analysis script needs a transient transformation — subsample, on-the-fly variable construction, reshaping for one estimation, dropping observations for a robustness cut — it wraps the transformation so the in-memory dataset is restored to the as-loaded state before the next estimation runs. Stata: `preserve` / `restore` for short scopes; `frames` for multi-dataset work. R/Python: build a new in-memory object (`data_sub <- ...`, `df_sub = ...`); never mutate the canonical data frame in place across estimations.

## Talk

**INV-20.** Notation in talk matches paper exactly — same symbols, same subscripts, same definitions.

**INV-21.** Every claim on a slide is traceable to the paper. No orphan results or numbers that don't appear in the manuscript.

---

## How Agents Use This File

| Agent | Checks | Action on Violation |
|-------|--------|-------------------|
| **writer-critic** | INV-1 through INV-13 | Deduct per scoring rubric |
| **coder-critic** | INV-13 through INV-19, INV-22, INV-23 | Deduct per scoring rubric |
| **storyteller-critic** | INV-20, INV-21 | Deduct per scoring rubric |
| **verifier** | INV-9, INV-10, INV-14, INV-15, INV-16, INV-19 | FAIL if present |
| **lint hook** | INV-14, INV-15, INV-16, INV-19, INV-22, INV-23 | Advisory warning |
