# Plan — Stata-Default Adaptation of clo-author Fork

**Status:** COMPLETED 2026-05-04 — all 11 execution steps verified; 4 carry-over items persisted to MEMORY.md under "Future TODO — clo-author fork housekeeping (added 2026-05-04)". **Post-execution patch (same date):** Stata path-globals convention corrected from upper-case (`$ROOT/$DATA/$WORK/$TAB/$FIG`) to lowercase (`$root/$rawdata/$workingdata/$tempdata/$figure/$table`) per user preference; `version 17` → `version 18`. References below updated for accuracy.
**Date:** 2026-05-04
**Author:** Hongbo Li (with Claude Code)
**Slug:** `stata-adaptation`

---

## Context

This repo (`~/claude-workflow/`) is my fork of `hugosantanna/clo-author`, a Claude Code scaffold for empirical economics research. Upstream is **R-default**: agents (`coder`, `coder-critic`, `data-engineer`), the `analyze` skill, content rules, and the verifier all assume R is the analysis language unless otherwise specified.

My methodological DNA (per `~/.claude/CLAUDE.md`) is **Stata > Python > R**:
- Stata via `stata-mcp` at `localhost:4000` for estimation, tables (`esttab`/`estout`), and most empirical figures (`graph`, `binsreg`, `coefplot`)
- Python (`polars`, `matplotlib`/`seaborn`) for wrangling, structural estimation, and figures when Stata is awkward
- MATLAB also used for structural estimation (deferred — flagged but not addressed tonight)
- R is fine as a fallback but should not be the default routing target

**Goal of this session:** the *minimum* set of edits that makes this fork usable for a Stata-first paper, while preserving everything clo-author does well (worker-critic pairs, quality gates, journal targeting, AEA replication compliance, R&R routing). No new project-specific names or data — the fork stays public-friendly.

**Constraint reminder:** This is the LIBRARY, not a specific paper. The current `CLAUDE.md` template uses bracketed placeholders for project info; we will not fill them with a fake project. Instead, `CLAUDE.md` becomes a library-level config that documents this fork's conventions.

**Out of scope tonight:** new agents or skills, MATLAB integration, structural-estimation Python conventions, deep AEA-package Stata audits.

---

## A. CLAUDE.md Library Rewrite

The current `CLAUDE.md` (lines 1–8) opens with `**Project:** [YOUR PROJECT NAME]` etc. — a project template, not a library config. Rewrite it as a library-level file.

**Keep:**
- Core Principles section (plan-first, verify-after, single source of truth, quality gates, worker-critic pairs, auto-memory)
- Folder Structure block (template paths users will copy)
- Commands section (latexmk usage)
- Quality Thresholds table
- Skills Quick Reference table
- Output Organization line

**Replace:**
- Header block (lines 3–6): swap `**Project:** [YOUR PROJECT NAME]` etc. for a Library Identity block — name the upstream (`hugosantanna/clo-author`), my fork purpose (Stata-default empirical labor adaptation), a one-line note that downstream users replace this header when they fork into a real project.
- Getting Started (lines ~17–21): replace step 1 ("fill in bracketed placeholders") with a forking note — derived projects copy this `CLAUDE.md` into their project root and fill the project header there.
- "Current Project State" table at the bottom: remove (it's project-specific) or convert to a "Library Reference State" stub that lists upstream version + fork divergence.

**Add:**
- A new **Analysis Language** subsection under Core Principles (or as its own short section): one paragraph documenting the fork's language priority — *Stata default for estimation, tables, and most figures; Python for wrangling and select figures (esp. structural); R retained as supported but secondary; MATLAB noted but not yet wired in*. Point to `.claude/references/coding-standards-stata.md` (the new reference file in §C).
- A pointer to the existing rules referenced by this fork (`.claude/rules/`) so a forker knows to read them.

**Tone:** terse, library-flavored, no project-specific naming. Public-friendly — anyone forking my fork should understand the Stata bias from one read.

---

## B. Stata-Aware Settings.json Permissions

Current `.claude/settings.json` allows `Bash(Rscript *)` and `Bash(python3 *)` but nothing for Stata. Per my workflow, Stata runs through MCP, not Bash piping `.do` files.

**Additions to `permissions.allow`:**

1. **MCP wildcard for stata-mcp** — pattern likely `mcp__stata_mcp__*` (exact MCP server name to be confirmed when stata-mcp is registered at the Claude Code level; I'll verify the connector name before editing). This is the canonical execution path.
2. **Bash fallback for batch Stata** — `Bash(stata-mcp *)` for any CLI helpers; `Bash(stata-se -b *)` and `Bash(stata-mp -b *)` only as escape hatches, not preferred.
3. **Python expansions** — `Bash(python *)` (not just `python3`), `Bash(jupyter *)`, `Bash(jupyter nbconvert *)`. Useful for the polars/matplotlib path.
4. **Stata-friendly file types** — none needed at the permissions level; Edit/Write already cover `.do`, `.ado`, `.dta` is binary so we wouldn't edit it.

**Cleanups (optional, low-priority):**
- The current `allow` list has stale entries from an earlier session: `Bash(do if:*)`, `Bash(then echo:*)`, `Bash(fi)`, the literal `find ~/Desktop -iname *ABDC*`, and `Edit(/.claude/skills/review/**)` with a leading slash that won't match. These are harmless but ugly. Defer cleanup unless it gets in the way.
- `additionalDirectories` points to `/Users/hsantanna/repos/clo-author/...` — that's the upstream maintainer's machine path, not mine. Either remove or replace with `${CLAUDE_PROJECT_DIR}/.claude/rules/archive` once I confirm the hooks need it. Defer.

**Hooks block:** untouched — `protect-files.sh`, `pre-compact.py`, `post-compact-restore.py`, `post-edit-lint.sh` are all language-agnostic infrastructure. Leave as-is.

---

## C. Highest-Leverage Files to Edit (5 ranked)

The two parallel Explore-agent surveys converged on the same hotspots. Ranked by **bang-for-buck** — leverage divided by edit cost:

### 1. `.claude/skills/analyze/SKILL.md` (HIGHEST LEVERAGE)
- **Lines 60–68 + 107–138 + 192–193:** R-coupled defaults baked into the default analyze flow — `scripts/R/`, `library(tidyverse)`/`fixest`/`modelsummary`/`ggplot2`, mandatory `saveRDS()`, `.rds` everywhere.
- **Edit:** rewrite the "Coder follows these principles" block (lines 62–68), the Script Structure Template (lines 106–138), and the Principles "saveRDS everything" line (193) to be language-aware. Default routing → Stata; secondary → Python; tertiary → R. Replace the `saveRDS` mandate with a language-agnostic "save intermediate objects" rule that lists per-language conventions: `.dta` / `estimates save` / `tempfile` for Stata; `.parquet` or `.pkl` for Python; `.rds` for R.
- **Why first:** the Coder agent reads this skill on every analysis run. One file edit changes routing for every future project without touching the agent definitions.

### 2. `.claude/agents/coder.md`
- **Lines 28–56:** "Default to R if not specified", `scripts/R/` layout, `.R` function-per-file, R-package laundry list.
- **Edit:** flip line 28 to `Default to Stata if not specified. Supported alternatives: Python, R.`. Add a Stata layout sibling block alongside `scripts/R/` (or replace it with a templated layout that branches on `${LANG}`). Add a pointer at line 30–33 to the new `coding-standards-stata.md`. Leave the design-specific package list (lines 114–135) but add a Stata column or a parallel "Stata equivalents" mini-table — mapping `fixest::feols` → `reghdfe`, `did::att_gt` → `csdid`, `rdrobust` (R) → `rdrobust` (Stata, same name), `Synth` → `synth` / `synth_runner`, etc.
- **Why second:** strategist.md is language-neutral; coder.md is the next layer where language assumptions concentrate.

### 3. `.claude/references/coding-standards-stata.md` (NEW FILE — the only one user authorized)
- Mirror the structure of the existing R/Python/Julia reference files (read one of them first to match form).
- Sections to include: file layout (`scripts/stata/`, master `.do` numbering), reproducibility (`version 18`, `set seed`, `set type double`), path discipline (project-root macros via `_setup.do` defining `$root`, `$rawdata`, `$workingdata`, `$tempdata`, `$figure`, `$table` per my project preferences — lowercase paths, uppercase only for tunable constants like `$SEED`/`$N_BOOT`), prohibited patterns (no `cd`, no global `clear all` mid-pipeline, no hardcoded paths), preferred packages (`reghdfe`, `ivreghdfe`, `boottest`, `did_imputation`, `csdid`, `rdrobust`, `binsreg`, `coefplot`, `esttab`/`estout`), table export patterns (bare `tabular` via `esttab` with `booktabs` rules, no `\begin{table}` wrapper per INV-13), figure export patterns (`graph export FILENAME.pdf, replace`).
- **Why third:** unlocks the other edits — coder.md and coder-critic.md both reference language-specific standards. Without this file, the Stata branch in those agents has nothing to point to.

### 4. `.claude/rules/content-invariants.md` — INV-16 only (line 41)
- Current text: `INV-16. No absolute paths. All paths relative to project root via here() (R), pathlib.Path (Python), or joinpath(@__DIR__, ...) (Julia).`
- **Edit:** add Stata: paths via project-root macros set in a central `_setup.do` (e.g., `$root`, `$rawdata`, `$workingdata`, `$tempdata`, `$figure`, `$table`). No `cd` outside `_setup.do`.
- **Why fourth:** without this, the verifier (which enforces INV-16, per content-invariants.md line 91 and verifier.md line 12) will FAIL on every Stata project. Single-line edit, blocks otherwise-passing audits.

### 5. `.claude/agents/verifier.md` (lines 38–46, 60–67, 86–92)
- **Line 40** hardcodes `Rscript scripts/R/FILENAME.R` for script execution check.
- **Line 66** treats `renv.lock` as the canonical R dependency manifest; no Stata equivalent listed.
- **Lines 86–92** describe AEA README format generically but the implicit examples are R-shaped.
- **Edit:** make the script-execution check language-aware — detect `.do` files and route through stata-mcp (or `stata -b do`) instead of `Rscript`. Add a Stata branch in Dependency Verification — Stata has no lockfile, so document `version` directives at the top of master scripts plus `ssc install` instructions in the README. Touch up the AEA README section to mention Stata package manifests explicitly.

---

### Honorable mentions — touched only if quick

- **`.claude/agents/coder-critic.md`** — language-specific check categories (lines 83–125) embed R idioms. Once `coding-standards-stata.md` exists, add a one-line redirect: "When the project language is Stata, apply the corresponding rules from `coding-standards-stata.md` instead of the R checks below." Don't rewrite the R checklist tonight — leave it intact for R projects.
- **`.claude/agents/data-engineer.md`** — line 30, lines 79–95 are heavily R/`ggplot2`. Add a one-line note that figure code defaults to Stata graph commands; when Python is selected, use matplotlib/seaborn. Defer the deeper rewrite.
- **`.claude/rules/content-standards.md`** lines 137–170 — the table conventions only document `modelsummary`, `fixest::etable`, `kableExtra`. Append a Stata subsection showing `esttab` exporting bare `tabular` with `booktabs`. Important for INV-1, INV-3, INV-13 compliance from Stata. Quick edit; consider for tonight.

---

## D. What NOT to Touch Tonight

Preserve until I have a real Stata project to stress-test against:

- `.claude/agents/strategist.md` and `strategist-critic.md` — language-neutral by design; no edits needed.
- `.claude/agents/theorist.md` / `theorist-critic.md` — proof-level, language-irrelevant.
- `.claude/agents/writer.md` / `writer-critic.md` — LaTeX-focused; Stata-Python-R indifferent.
- `.claude/agents/storyteller.md` / `storyteller-critic.md` — Beamer/Quarto talks, no analysis-language coupling.
- `.claude/agents/librarian.md`, `explorer.md`, `editor.md`, `domain-referee.md`, `methods-referee.md`, `orchestrator.md` — none touch analysis language.
- `.claude/skills/write/`, `submit/`, `review/`, `revise/`, `talk/`, `discover/`, `strategize/`, `tools/`, `checkpoint/`, `new-project/` — none of the surface checks I'd want for tonight; the Stata switch goes through `analyze` and the agents below it.
- `.claude/rules/working-paper-format.md` — LaTeX preamble standard; language-agnostic.
- `.claude/rules/quality.md`, `agents.md`, `workflow.md`, `meta-governance.md`, `revision.md`, `logging.md` — pipeline rules, no language coupling.
- `.claude/references/coding-standards-r.md`, `coding-standards-python.md`, `coding-standards-julia.md` — leave intact; they remain valid for R/Python/Julia branches.
- `.claude/references/journal-profiles.md`, `domain-profile.md` — domain-level, not language-level.
- `.claude/hooks/*.sh|*.py` — infrastructure; untouched.
- `CHANGELOG.md`, `README.md` — defer until the adaptation actually lands and is tested. (We'll bump the changelog on the commit, not in the plan.)
- `paper/`, `data/`, `scripts/`, `templates/`, `master_supporting_docs/`, `explorations/` — content directories, untouched.
- MATLAB integration — flagged in `CLAUDE.md` as "noted but not wired"; no agent or rule edits tonight.
- The cosmetic settings.json cleanups (stale allow entries, `additionalDirectories` referencing `hsantanna`) — defer.

---

## E. Order of Operations (with Verification)

Sequencing matters: the new reference file must exist before agents point to it; INV-16 must accept Stata paths before verifier runs.

| # | Edit | File | Verification |
|---|------|------|--------------|
| 1 | Author Stata coding-standards reference | `.claude/references/coding-standards-stata.md` (NEW) | Read it back end-to-end; check it mirrors `.claude/references/coding-standards-r.md` structure; confirm it covers reproducibility, paths, packages, table export, figure export, prohibited patterns. |
| 2 | Extend INV-16 to Stata | `.claude/rules/content-invariants.md` line 41 | `grep -n "INV-16" .claude/rules/` to confirm only one location updated; re-read the line. |
| 3 | Library rewrite of `CLAUDE.md` | `/Users/hongboli/claude-workflow/CLAUDE.md` | Diff against pre-edit; confirm no project-specific names; confirm Stata > Python > R priority is documented; confirm folder-structure block still present. |
| 4 | Settings.json permissions additions | `.claude/settings.json` | `cat .claude/settings.json \| python3 -m json.tool` to validate JSON; confirm new entries appear in `allow`; restart Claude Code session and confirm no permission prompts on a trial Stata MCP call (do this manually after the session). |
| 5 | Default-language flip in coder.md | `.claude/agents/coder.md` lines 28–56 | Re-read lines 28–56; confirm Default = Stata; confirm pointer to `coding-standards-stata.md`; confirm a Stata layout block exists alongside R. |
| 6 | analyze/SKILL.md language-awareness | `.claude/skills/analyze/SKILL.md` lines 60–68, 107–138, 192–193 | Re-read modified blocks; confirm `.dta` / `estimates save` for Stata; confirm Script Structure Template now branches by language or is presented as Stata-default with R/Python siblings; confirm "saveRDS everything" replaced with a language-agnostic equivalent. |
| 7 | verifier.md language-aware execution | `.claude/agents/verifier.md` lines 38–46, 60–67 | Re-read lines; confirm `.do` execution path documented; confirm Stata dependency-manifest guidance added. |
| 8 *(stretch)* | content-standards.md Stata table subsection | `.claude/rules/content-standards.md` after line 170 | Confirm new subsection shows `esttab` exporting bare `tabular` with `booktabs`, references INV-1, INV-3, INV-13. |

**End-of-session verification (single pass):**

1. `grep -rni "Default to R" .claude/agents/ .claude/skills/ .claude/rules/` — should return zero hits (or only inside a comment that explicitly shows the R fallback is secondary).
2. `grep -rn "stata" .claude/` — confirm new content lands where expected; spot-check no stray case errors.
3. `python3 -m json.tool .claude/settings.json` — JSON valid.
4. Re-read the rewritten `CLAUDE.md` cold — does it read as a library, not a half-filled project template? No bracketed `[YOUR …]` placeholders left.
5. Open this plan, mark Status: COMPLETED with a one-line summary and timestamp.
6. Append a SESSION_REPORT.md entry per `.claude/rules/logging.md` and a research_journal.md entry.
7. Commit message: `Stata-default adaptation (CLAUDE.md library rewrite + 5 high-leverage edits)` — single commit, public-friendly.

**Non-goals for end-of-session verification:**

- Running an actual Stata script through the verifier — defer to first real project.
- Testing stata-mcp permissions in practice — defer to first MCP call (will surface naturally).
- Confirming AEA replication audit works for `.do` files — defer; will need a real package to audit.

---

## Critical Files Snapshot

| File | Role | Touched tonight? |
|------|------|------------------|
| `/Users/hongboli/claude-workflow/CLAUDE.md` | Library config | YES — full rewrite |
| `/Users/hongboli/claude-workflow/.claude/settings.json` | Permissions + hooks | YES — additions only |
| `/Users/hongboli/claude-workflow/.claude/skills/analyze/SKILL.md` | Analysis dispatch skill | YES — sections 62–68, 107–138, 192–193 |
| `/Users/hongboli/claude-workflow/.claude/agents/coder.md` | Coder agent | YES — lines 28–56, package map |
| `/Users/hongboli/claude-workflow/.claude/agents/verifier.md` | Verification agent | YES — lines 38–46, 60–67 |
| `/Users/hongboli/claude-workflow/.claude/rules/content-invariants.md` | Invariants | YES — INV-16 only |
| `/Users/hongboli/claude-workflow/.claude/references/coding-standards-stata.md` | Stata coding standards | NEW — created tonight |
| `/Users/hongboli/claude-workflow/.claude/agents/coder-critic.md` | Code review | STRETCH — one-line redirect |
| `/Users/hongboli/claude-workflow/.claude/agents/data-engineer.md` | Data + figures | STRETCH — one-line note |
| `/Users/hongboli/claude-workflow/.claude/rules/content-standards.md` | Tables/figures conventions | STRETCH — `esttab` subsection |

---

## Open Questions / Risks

> Note: the original Open Question #1 (stata-mcp connector name) was resolved during execution per the user's OVERRIDE 3 — the canonical pattern is `mcp__stata-mcp__*` (hyphenated, per Claude Code's `mcp__<server-name>__<tool-name>` convention). The four items below were persisted to MEMORY.md under "Future TODO — clo-author fork housekeeping (added 2026-05-04)" so they survive into future maintenance sessions.

1. **MATLAB.** Deferred — `CLAUDE.md` mentions MATLAB as a known structural-estimation tool but no skill, agent, or settings work happened in this session. Track as future TODO.
2. **R retention.** Existing R coding-standards reference and the R-shaped sections in `coder.md`, `coder-critic.md`, `content-standards.md` stay intact for users still running R. The fork is Stata-default but not Stata-only.
3. **Coverage of `data-engineer.md`.** Punted the deep refactor; the figure-toolkit block-quote note was added at the top of §2, but the package table and `ggplot2` theme block below it are still R-flavored. Real overhaul waits for a project that actually needs it.
4. **Settings.json hook paths.** The `additionalDirectories` entry points to the upstream maintainer's home directory. Touching this without verifying the hooks could break post-edit-lint. Defer cleanup; document in CHANGELOG when the adaptation lands.
