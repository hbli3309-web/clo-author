# CLAUDE.md — clo-author Fork: Stata-Default Empirical Economics

<!-- This is the LIBRARY-level config for this fork of hugosantanna/clo-author.
     Adapted to Stata-first empirical labor economics. The file is loaded by
     Claude Code every session.

     If you are forking THIS repo into a real project, replace this header
     block with your project's identity (project name, institution, field,
     branch). The rest of this file documents the fork's conventions and is
     useful as-is.

     Keep this file under ~150 lines.
     Upstream guide: https://hugosantanna.github.io/clo-author/ -->

**Library:** `claude-workflow` (fork of `hugosantanna/clo-author`)
**Maintainer:** Hongbo Li
**Adaptation:** Stata-default empirical labor economics; preserves clo-author's worker-critic pairs, quality gates, journal targeting, AEA replication compliance, and R&R routing
**Branch:** main

---

## Core Principles

- **Plan first** — enter plan mode before non-trivial tasks; save plans to `quality_reports/plans/`
- **Verify after** — compile and confirm output at the end of every task
- **Single source of truth** — paper `main.tex` is authoritative; talks and supplements derive from it
- **Quality gates** — weighted aggregate score; nothing ships below 80/100; see `.claude/rules/quality.md`
- **Worker-critic pairs** — every creator has a paired critic; critics never edit files
- **Auto-memory** — corrections and preferences are saved automatically via Claude Code's built-in memory system

---

## Analysis Language Priority

This fork is **Stata-default**. The language priority is:

| Rank | Language | Use Case | Reference |
|------|----------|----------|-----------|
| 1 | **Stata** | Reduced-form estimation, tables (`esttab`/`estout`), most empirical figures (`graph`/`binsreg`/`coefplot`) | `.claude/references/coding-standards-stata.md` |
| 2 | **Python** | Wrangling (`polars`, `pandas`), structural estimation, figures when Stata is awkward | `.claude/references/coding-standards-python.md` |
| 3 | **R** | Retained as supported but secondary; useful when a published estimator is R-only | `.claude/references/coding-standards-r.md` |
| — | **MATLAB** | Noted for structural estimation; not yet wired into the agent pipeline | future TODO |

Stata is invoked through the **`stata-mcp` MCP server**, not by piping `.do` files via `Bash`. Project's local `CLAUDE.md` may override this priority.

---

## Getting Started (when forking this fork into a project)

1. Copy this `CLAUDE.md` into your new project root, then replace the **Library / Maintainer / Adaptation / Branch** header block with your project's identity.
2. Read the rules in `.claude/rules/` — they are non-negotiable engineering and content invariants.
3. Run `/discover interview [topic]` to build your research specification, or `/new-project [topic]` for the full orchestrated pipeline.

---

## Folder Structure

```
project-root/
├── CLAUDE.md                    # This file
├── .claude/                     # Rules, agents, skills, references, hooks
│   ├── rules/                   # Engineering and content invariants (READ THESE)
│   ├── agents/                  # Worker and critic agent definitions
│   ├── skills/                  # Slash-command skill packs
│   └── references/              # Domain profile, journal profiles, coding standards
├── Bibliography_base.bib        # Centralized bibliography
├── paper/                       # Main LaTeX manuscript (source of truth)
│   ├── main.tex                 # Primary paper file
│   ├── sections/                # Section-level .tex files
│   ├── figures/                 # Generated figures (.pdf, .png)
│   ├── tables/                  # Generated tables (.tex — bare tabular)
│   ├── talks/                   # Beamer presentations
│   ├── quarto/                  # Quarto RevealJS presentations
│   ├── preambles/               # LaTeX headers / shared preamble
│   ├── supplementary/           # Online appendix and supplements
│   └── replication/             # Replication package for deposit
├── data/                        # Project data
│   ├── raw/                     # Original untouched data (often gitignored)
│   └── cleaned/                 # Processed datasets ready for analysis
├── scripts/                     # Analysis code
│   ├── stata/                   # Default: .do files, _setup.do, master.do, ado/
│   ├── python/                  # When using Python
│   └── R/                       # When using R
├── quality_reports/             # Plans, session logs, reviews, scores
├── explorations/                # Research sandbox
├── templates/                   # Session log, quality report templates
└── master_supporting_docs/      # Reference papers and data docs
```

---

## Commands

```bash
# Paper compilation (latexmk handles multi-pass + biber automatically)
cd paper && latexmk main.tex

# Talk compilation
cd paper/talks && latexmk talk.tex

# Clean auxiliary files
cd paper && latexmk -c
```

> **Note:** `paper/latexmkrc` configures XeLaTeX, TEXINPUTS, BIBINPUTS. On Overleaf, set compiler to XeLaTeX — Overleaf reads `latexmkrc` automatically.

---

## Quality Thresholds

| Score | Gate | Applies To |
|-------|------|------------|
| 80 | Commit | Weighted aggregate (blocking) |
| 90 | PR | Weighted aggregate (blocking) |
| 95 | Submission | Aggregate + all components >= 80 |
| — | Advisory | Talks (reported, non-blocking) |

See `.claude/rules/quality.md` for the weighted aggregation formula.

---

## Skills Quick Reference

| Command | What It Does |
|---------|-------------|
| `/new-project [topic]` | Full pipeline: idea → paper (orchestrated) |
| `/discover [mode] [topic]` | Discovery: interview, literature, data, ideation |
| `/strategize [mode] [question]` | Identification strategy, pre-analysis plan, or formal theory (`theory` mode) |
| `/analyze [dataset]` | End-to-end data analysis (Stata default) |
| `/write [section]` | Draft paper sections + cleanup pass (`style-guide` extracts voice) |
| `/review [file/--flag]` | Quality reviews (routes by target: paper, code, peer) |
| `/revise [report]` | R&R cycle: classify + route referee comments |
| `/talk [mode] [format]` | Create, audit, or compile Beamer presentations |
| `/submit [mode]` | Journal targeting → package → audit → final gate |
| `/tools [subcommand]` | Utilities: commit, compile, validate-bib, journal, etc. |
| `/checkpoint [--flag]` | Session handoff: memory + SESSION_REPORT + research journal |

---

## Output Organization

<!-- Options: by-script (default) or by-purpose
     by-script:  paper/figures/04_estimation/coefplot.pdf
     by-purpose: paper/figures/estimation/coefplot_main.pdf -->
Output organization: by-script

---

## Library Reference State

This is the upstream library, not a paper. Downstream projects forking this `CLAUDE.md` should replace this section with their own paper / data / replication / talk status table.
