# Coding Standards: Stata

These standards apply to all Stata code produced by the Coder agent. Adapted from C++ Core Guidelines engineering discipline for Stata in empirical economics. The coder-critic enforces these rules. Companion to `coding-standards-r.md` and `coding-standards-python.md`.

---

## 1. Runtime and Dependencies

- **Stata/MP 18+** is the project default (Stata/SE 18+ acceptable; Stata/MP gets parallel speedups on `reghdfe`)
- **`version` directive** at the top of every master `.do` file (`version 18`) — pins behavior across Stata releases
- **No on-the-fly installs in scripts.** `ssc install` lives in a separate documented setup script (or in the README); never `ssc install` inline in `01_setup.do` or any analysis `.do`
- **Execution path:** Stata is invoked through the `stata-mcp` MCP server (preferred), not by piping `.do` files through `Bash`. The MCP server holds session state, which matters because Stata is single-dataset-in-memory

### Core Stack

| Command / Package | Purpose |
|-------------------|---------|
| `reghdfe` | High-dimensional fixed effects (replaces `xtreg, fe absorb()`) |
| `ivreghdfe` | IV with high-dim FE |
| `boottest` | Wild-cluster bootstrap inference |
| `did_imputation`, `csdid`, `eventdd` | Robust DiD / event study (Borusyak-Jaravel-Spiess, Callaway-Sant'Anna) |
| `rdrobust`, `rdmulti`, `rddensity` | RDD estimation, density tests |
| `binsreg` | Binned scatter regressions |
| `coefplot` | Coefficient plots |
| `esttab` / `estout` | LaTeX table export |
| `synth`, `synth_runner` | Synthetic control |
| `gtools` | Fast `gegen` / `gcollapse` / `gquantiles` (drop-in replacements) |
| `frames` (built-in 16+) | Multi-dataset workflows without `preserve`/`restore` thrash |

### Prohibited / Legacy

| Command | Reason | Replacement |
|---------|--------|-------------|
| `xtreg, fe absorb()` | Cannot do 2+ way HDFE | `reghdfe` |
| `xi:` prefix | Deprecated; loads memory with explicit dummies | factor-variable notation `i.var` |
| Manual `forvalues` over panel IDs | Slow, error-prone | `bysort` / `egen` / `frames` |
| `set more off` left in scripts | Sticky setting, surprises later sessions | put `set more off` in `_setup.do` only |

### Table-Export Tools — Both Supported

`esttab` / `estout` is the **preferred** path for AER-style three-line `booktabs` tables — its `booktabs fragment` mode produces bare `tabular` output that drops directly into `main.tex` (per INV-13). `outreg2` is also supported as a legitimate alternative; it does not emit `booktabs` rules natively, so for AER-style submission either post-process the output or prefer `esttab`. Use `outreg2` when its summary-statistics or panel-side ergonomics are a better fit.

---

## 2. Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Files | `snake_case.do` with numeric prefix | `01_setup.do`, `02_clean.do`, `04_estimation.do` |
| Macros (locals) | `snake_case` | `local n_obs`, `local cluster_var` |
| Macros (globals — paths) | `lowercase` | `$root`, `$rawdata`, `$workingdata`, `$tempdata`, `$figure`, `$table` |
| Macros (globals — tunable constants) | `UPPER_SNAKE_CASE` | `$SEED`, `$N_BOOT` |
| Variables | `snake_case`, lowercase | `treatment`, `log_income`, `cluster_id` |
| Programs / ado | `snake_case`, verb prefix | `estimate_att`, `test_oir` |
| Loop indices | Short, scoped | `i`, `j`, `b` (bootstrap), `g` (group), `t` (time) |
| Booleans | `is_`, `has_` prefix | `is_treated`, `has_match` |
| Stored estimates | `model_<spec>` | `eststo model_baseline`, `eststo model_iv` |

**All commands lowercase.** `regress`, not `REGRESS` or `Regress`. Stata is case-sensitive for variable names; commands are case-insensitive but lowercase is the universal convention.

---

## 3. Code Style

- **Line width:** 100 characters soft cap (Stata gets verbose with options); break long commands with `///`
- **Indentation:** 4 spaces inside `forvalues` / `foreach` / `program` blocks (no tabs)
- **Comments:** `*` for whole-line, `//` for end-of-line, `/* ... */` for block. Document the *why*, not the *what*
- **Continuation:** `///` at end of line — never bare backslash
- **Blocks:** opening `{` on same line as command; closing `}` on its own line
- **Quotes:** prefer compound `` `"..."' `` for any string that may contain quotes
- **No trailing whitespace.** Stata's `do`-file editor strips it; external editors should too

```stata
* GOOD
reghdfe outcome treatment $controls,                                ///
    absorb(unit_id year)                                             ///
    cluster(state_id)
```

---

## 4. Numerical Discipline

The C++ principle: **never trust implicit behavior with numbers.**

### Float Comparison
```stata
* WRONG
if `x' == 0.1 + 0.2 ...

* RIGHT — use a tolerance
if abs(`x' - 0.3) < 1e-10 ...
* or for matrices:
if mreldif(A, B) < 1e-10 ...
```

### CDF / Probability Clamping
```stata
* CDF values must stay in [0, 1]
gen f_hat = max(0, min(f_hat_raw, 1))
```

### Inverse Link Guards
```stata
* Guard against invnormal(0) = -infinity, invnormal(1) = +infinity
local eps = 1e-12
gen p_safe = max(`eps', min(1 - `eps', p))
gen z = invnormal(p_safe)
```

### Integer Discipline
- Stata stores numbers as `float` by default — **set `set type double` in `_setup.do`** for precision-sensitive work (matching estimators, EM iterations, GMM)
- Use `int()`, `round()`, or `floor()` explicitly when you mean an integer; never rely on coercion
- Loop bounds: `forvalues b = 1/`N_BOOT'` is safe; bare `1/N` reads `N` as a global only if it exists

### Missing Values
- Stata missings (`.`) are **larger than any number** in comparisons. `gen high = (income > 100000)` will set `high = 1` for missing `income`. Always guard:
  ```stata
  gen high = (income > 100000) & !missing(income)
  ```
- After every merge, `tabulate _merge` and document master-only / using-only counts

### Pre-allocation (Mata / matrix work)
```stata
* RIGHT — declare matrix size up-front
matrix boot_results = J(`n_grid', `N_BOOT', .)
forvalues b = 1/`N_BOOT' {
    matrix boot_results[1, `b'] = ...
}
```

---

## 5. Function Design (Programs and Ado Files)

### One Program Per File
Reusable programs live in `scripts/stata/ado/` (or a project `ado` directory added to `adopath`). File name matches program name: `estimate_att.ado` contains `program define estimate_att`.

### Consistent API
```stata
program define estimate_att, rclass
    syntax varlist(min=2 max=2) [if] [in],         ///
        Cluster(varname)                           ///
        [Bootstrap(integer 0)]
    
    * Preconditions
    confirm numeric variable `varlist'
    marksample touse
    
    * Implementation
    quietly reghdfe `varlist' if `touse', cluster(`cluster')
    
    * Return scalars
    return scalar att = _b[treatment]
    return scalar se  = _se[treatment]
    return scalar n   = e(N)
end
```

### Fail Fast
```stata
assert !missing(unit_id) & !missing(time)
confirm numeric variable outcome
if _N == 0 {
    error 2000  // no observations
}
```

### Help Files
Every reusable `.ado` ships with a `.sthlp` documenting `Description`, `Syntax`, `Options`, `Returned values`. The coder-critic checks for this.

---

## 6. Data Idioms

- **`tempfile` for intermediates** — never write `.dta` files for in-pipeline intermediates that aren't needed downstream:
  ```stata
  tempfile cleaned
  save `cleaned', replace
  ```
- **`preserve`/`restore` is required around any analysis-stage mutation** of the loaded dataset (subsample, on-the-fly variable construction, reshape, drop) — see Section 14 (Spec Discipline). Use `frames` instead when you genuinely need two datasets simultaneously (Stata 16+); avoid serial `preserve`/`restore` chains, but never let a `keep`/`drop`/`replace` leak into the next estimation's sample.
- **Merges:** always `assert _merge ...` after, document expected master-only / using-only counts. Drop `_merge` before next merge
- **`gen` vs `egen`:** use `egen` for group operations and aggregations; reaching for `egen` to do a simple transform is overkill
- **I/O:** `use`, `save`, `merge`, `import delimited` / `export delimited`. Parquet via `parquet` (community package) when interoperating with Python/polars

---

## 7. Bootstrap and Parallelism

```stata
* Wild-cluster bootstrap via boottest
reghdfe outcome treatment $controls, absorb(unit_id year) cluster(state_id)
boottest treatment, reps($N_BOOT) cluster(state_id) seed(${SEED})
```

For block bootstrap or simulation:
- Set `set seed ${SEED}` once in `_setup.do` (after `version` directive)
- For Stata/MP, `reghdfe` and `ivreghdfe` parallelize automatically via `set processors`
- For embarrassingly parallel simulations, the `parallel` (community) package shells out to multiple Stata sessions; reserve for >5-minute runs and document the install in the README

---

## 8. Error Handling

- `assert <condition>` — hard fails on violation; preferred for invariants
- `confirm numeric variable X` / `confirm file Y` — type/file-existence preconditions
- `capture <command>` — only when you genuinely expect a tolerated failure (e.g., dropping a variable that may not exist); always check `_rc`:
  ```stata
  capture confirm variable old_name
  if _rc == 0 drop old_name
  ```
- Never silently swallow errors. Bare `capture` without `_rc` checks is a code smell

---

## 9. Path Discipline (INV-16 Compliance)

**Required:** every project sets project-root macros in a single `_setup.do` and references them everywhere.

```stata
* _setup.do — sourced by 00_master.do
version 18
clear all
set type double
set more off

global root         "`c(pwd)'"
global rawdata      "$root/data/raw"
global workingdata  "$root/data/cleaned"
global tempdata     "$root/data/temp"
global figure       "$root/paper/figures"
global table        "$root/paper/tables"

global SEED   20260504
global N_BOOT 1000
set seed $SEED
```

Every analysis `.do` references `$rawdata`, `$workingdata`, `$tempdata`, `$table`, `$figure`. **Never `cd` outside `_setup.do`.** The verifier enforces INV-16 against this.

---

## 10. Prohibited Patterns

| Pattern | Reason | Replacement |
|---------|--------|-------------|
| `cd <path>` outside `_setup.do` | Breaks portability and pipeline state | Project-root macros (`$root` etc.) |
| `clear all` mid-pipeline | Wipes globals and frames; surprises later steps | `clear` (data only) at script start |
| Hardcoded absolute paths | Breaks portability | `$root`, `$rawdata`, `$workingdata`, `$tempdata`, `$figure`, `$table` |
| `set more off` in analysis scripts | Sticky setting; should live in `_setup.do` | `_setup.do` only |
| `xi:` prefix | Deprecated | Factor-variable notation `i.var` |
| `outreg2` | Legacy LaTeX | `esttab` / `estout` |
| `xtreg, fe absorb()` for 2+ FE | Cannot do high-dim FE properly | `reghdfe` |
| `keep if` followed by `drop` | Confusing; double-restriction is hard to audit | One `keep if` with combined condition |
| `display "hello"` for status | Pollutes output | `noisily display as text "..."` only when intentional |
| Bare `capture` with no `_rc` check | Silent failures | `capture ...` followed by `if _rc != 0 ...` |
| `replace` after `gen` to fix logic | Edits are hard to audit | One `gen ... = cond(...)` instead |
| `egen, group()` for IDs you'll merge on | Order-dependent, breaks reproducibility | Hash from underlying keys |

---

## 11. Tables and Figures

### Tables — `esttab` exports bare `tabular` (per INV-13)

```stata
* Build the model collection
eststo clear
eststo model_ols: reghdfe outcome treatment $controls, absorb(unit_id year) cluster(state_id)
eststo model_iv:  ivreghdfe outcome (treatment = instrument) $controls, ///
    absorb(unit_id year) cluster(state_id)

* Export bare tabular (no \begin{table}, no caption, no notes — those go in main.tex)
esttab model_ols model_iv using "$table/main_results.tex",                ///
    replace booktabs fragment                                              ///
    b(3) se(3) star(* 0.10 ** 0.05 *** 0.01)                               ///
    keep(treatment) coeflabels(treatment "Treatment")                      ///
    stats(N r2_a, fmt(0 3) labels("Observations" "Adj. R$^2$"))           ///
    nomtitle nonotes nogaps
```

**For AEA journals:** drop `star(...)` — AEA style omits significance stars (per INV-4). Use confidence intervals with `cells("b ci")` instead.

The wrapping (`\begin{table}`, `\caption{}`, `threeparttable`, `\begin{tablenotes}`) happens in `paper/main.tex`. Generated `.tex` files contain `tabular` only.

### Figures — `graph export` to PDF

```stata
binsreg outcome treatment $controls, absorb(unit_id year)
graph export "$figure/binscatter_main.pdf", replace

coefplot model_ols model_iv, drop(_cons) keep(treatment $controls) ///
    xline(0) ciopts(recast(rcap))
graph export "$figure/coefplot_main.pdf", replace
```

**No titles inside the figure (per INV-12).** Titles go in LaTeX `\caption{}`. Panel labels for multi-panel figures via `graph combine`.

---

## 12. Reproducibility Checklist

Every project:

- [ ] `_setup.do` sets `version`, `set seed`, `set type double`, project-root globals
- [ ] `00_master.do` runs `_setup.do` first, then numbered scripts in order
- [ ] No `cd` outside `_setup.do`
- [ ] No `ssc install` inside analysis `.do` files (separate setup or README)
- [ ] All `merge` operations followed by `assert _merge ...` checks
- [ ] All bootstrapped / simulated estimates seeded
- [ ] `set processors` declared if using Stata/MP parallelism
- [ ] README documents Stata version, MP/SE/IC edition, package list (with version pins where possible)

---

## 13. Performance

- `gtools` (`gegen`, `gcollapse`, `gquantiles`) are 5–50× faster than the built-ins on large data; use them by default
- `frames` for multi-dataset workflows; avoid serial `preserve`/`restore` chains
- Stata is single-threaded outside Stata/MP commands — for embarrassingly parallel sims, use the `parallel` package or split runs across MCP sessions
- The bootstrap loop is almost always the bottleneck — `boottest` is wildly faster than naive bootstrap when applicable
- Profile with `set rmsg on` (per-command timing) before micro-optimizing

---

## 14. Spec Discipline and Analysis-Data Separation (INV-22, INV-23)

Two rules so that "let me try a different sample / outcome / window" is a one-line edit at the top of the script, and so that estimation N° 7 sees the same dataset as estimation N° 1.

### 14.1 Spec Macros — One Edit Surface (INV-22)

Every spec parameter that the user might want to vary is declared **once** as a named macro at the top of the analysis script, then referenced everywhere downstream. This covers, at minimum:

- Sample-selection filters (year range, age range, industry, geography)
- Outcome / dependent variable(s)
- Treatment definition
- Control set
- Fixed-effects set
- Cluster level
- Bandwidth (for RDD), reference period (for event study), event-time window

**Use `global` for cross-script spec** (set once in `_setup.do` or a project spec block), **`local` for script-scoped spec** (set at the top of an analysis `.do`).

```stata
*===============================================================
* 04_estimation.do — Main specification
*===============================================================
do "scripts/stata/_setup.do"

* ---- Spec block (edit here, not below) ----
local sample_if    "year >= 2010 & year <= 2019 & age >= 25 & age <= 60"
local outcome      log_wage
local treatment    treat_post
local controls     age age_sq i.educ i.industry
local fe           "absorb(worker_id year)"
local cluster_var  firm_id
* -------------------------------------------

use "$workingdata/analysis_panel.dta", clear

eststo clear
eststo m_baseline: reghdfe `outcome' `treatment' `controls' if `sample_if', ///
    `fe' cluster(`cluster_var')

esttab m_baseline using "$table/main_results.tex", replace booktabs fragment ///
    b(3) se(3) keep(`treatment') nomtitle nonotes nogaps
```

Equivalent with project-wide globals (use these when the same spec is shared across `04_estimation.do`, `05_robustness.do`, `06_figures.do`):

```stata
* in _setup.do or a dedicated _spec.do
global sample_if   "year >= 2010 & year <= 2019 & age >= 25 & age <= 60"
global outcome     log_wage
global treatment   treat_post
global controls    age age_sq i.educ i.industry
global cluster_var firm_id

* in 04_estimation.do
reghdfe $outcome $treatment $controls if $sample_if, absorb(worker_id year) cluster($cluster_var)
```

**Forbidden in analysis code:**

```stata
* WRONG — hardcoded sample filter, hardcoded outcome
keep if year >= 2010 & year <= 2019
reghdfe log_wage treat_post age age_sq i.educ, absorb(worker_id year) cluster(firm_id)
```

The `local` / `global` block at the top is the single edit surface. Robustness checks that vary one spec parameter override the relevant macro and rerun — they do not duplicate the spec inline.

### 14.2 Analysis-Data Separation + `preserve`/`restore` (INV-23)

Cleaning produces canonical cleaned data on disk. Analysis reads that data and never overwrites it. **Concretely:**

- `02_data_preparation.do` → writes `$workingdata/analysis_panel.dta` (and any analysis-ready derivatives)
- `04_estimation.do`, `05_robustness.do`, `06_figures.do` → `use "$workingdata/analysis_panel.dta", clear`; **never `save` back** to `$workingdata/`. Intermediates that the analysis itself creates go to `$tempdata/` or `tempfile`

Inside an analysis script you frequently need to mutate the loaded dataset for one estimation — drop observations for a subsample, build a one-off variable, reshape for an event-study, collapse for a cell-level regression. **Wrap the mutation in `preserve` / `restore`** so the next estimation sees the same as-loaded data:

```stata
use "$workingdata/analysis_panel.dta", clear

* Estimation 1 — full sample
eststo m_full: reghdfe `outcome' `treatment' `controls', `fe' cluster(`cluster_var')

* Estimation 2 — subsample (manufacturing only); restore after
preserve
    keep if industry == 31
    eststo m_mfg: reghdfe `outcome' `treatment' `controls', `fe' cluster(`cluster_var')
restore

* Estimation 3 — collapsed to firm-year for cell regression
preserve
    gcollapse (mean) `outcome' `treatment' `controls', by(firm_id year)
    eststo m_cell: reghdfe `outcome' `treatment' `controls', absorb(firm_id year) cluster(firm_id)
restore

* Estimation 4 — back on the original loaded sample
eststo m_alt: reghdfe `outcome' `treatment' alt_controls, `fe' cluster(`cluster_var')
```

When two datasets need to coexist (merging an aggregate-level panel with the individual-level one, or running an estimation on the cleaned data while keeping a different transform alive), use `frames` instead of nested `preserve`/`restore`:

```stata
frame copy default firm_year
frame firm_year: gcollapse (mean) `outcome' `treatment', by(firm_id year)
frame firm_year: reghdfe `outcome' `treatment', absorb(firm_id year) cluster(firm_id)
* default frame is untouched
```

**Forbidden in analysis code:**

```stata
* WRONG — mutates the in-memory dataset; estimation 2 sees a different sample
use "$workingdata/analysis_panel.dta", clear
reghdfe outcome treat $controls, absorb(unit year) cluster(state)
keep if industry == 31                          // bare keep — no preserve
reghdfe outcome treat $controls, absorb(unit year) cluster(state)

* WRONG — overwriting the cleaned file from an analysis script
save "$workingdata/analysis_panel.dta", replace
```

**Cleaning scripts** (`02_data_preparation.do` and friends) are the only place where mutation-then-save of the cleaned dataset is allowed. Analysis scripts are read-only with respect to `$workingdata/`.
