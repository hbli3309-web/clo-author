# ============================================================
# Claude Code 学术研究项目初始化函数
# 用法: new-paper <project-name> [--lang stata|python|r] [--no-git]
# 例: new-paper paper-trae-amplification
#     new-paper paper-driver-ai --lang python
# ============================================================
new-paper() {
  # ---------- 参数解析 ----------
  if [ $# -lt 1 ]; then
    echo "❌ Usage: new-paper <project-name> [--lang stata|python|r] [--no-git]"
    echo "   Example: new-paper paper-trae-amplification"
    return 1
  fi

  local PROJECT_NAME="$1"
  shift
  local PRIMARY_LANG="stata"   # 默认
  local INIT_GIT="yes"

  while [ $# -gt 0 ]; do
    case "$1" in
      --lang) PRIMARY_LANG="$2"; shift 2 ;;
      --no-git) INIT_GIT="no"; shift ;;
      *) echo "⚠️  Unknown flag: $1"; shift ;;
    esac
  done

  # ---------- 校验 ----------
  local LIBRARY_PATH="$HOME/claude-workflow"
  local PROJECT_PATH="$HOME/Research/$PROJECT_NAME"

  if [ ! -d "$LIBRARY_PATH" ]; then
    echo "❌ Library not found at $LIBRARY_PATH"
    echo "   Did you clone hb-workflow yet?"
    return 1
  fi

  if [ -d "$PROJECT_PATH" ]; then
    echo "❌ Project already exists at $PROJECT_PATH"
    echo "   Pick a different name, or rm -rf it first."
    return 1
  fi

  echo "📦 Initializing project: $PROJECT_NAME"
  echo "   Path:     $PROJECT_PATH"
  echo "   Language: $PRIMARY_LANG (primary)"
  echo ""

  # ---------- Step 1: clone 图书馆做骨架 ----------
  mkdir -p "$HOME/Research"
  cd "$HOME/Research" || return 1

  echo "🔄 Cloning library..."
  git clone --depth 1 "$LIBRARY_PATH" "$PROJECT_NAME" 2>&1 | tail -5
  cd "$PROJECT_NAME" || return 1

  # 切断跟图书馆的 git 关系
  rm -rf .git
  echo "✂️  Severed git history from library"

  # ---------- Step 2: 建项目级目录 ----------
  echo "📁 Creating project directories..."
  mkdir -p \
    data/{raw,cleaned,temp} \
    scripts/stata/ado \
    scripts/python/functions \
    scripts/R/functions \
    paper/{sections,figures,tables,talks,preambles,supplementary,replication} \
    explorations \
    quality_reports/{plans,session_logs,specs,checkpoints} \
    master_supporting_docs

  # ---------- Step 3: 写 Stata _setup.do ----------
  cat > scripts/stata/_setup.do << EOF
* ============================================================
* _setup.do — Project root configuration for $PROJECT_NAME
* All .do files must \`do _setup.do\` at the top
* ============================================================
version 18

* --- Project root ---
global root             "$PROJECT_PATH"

* --- Data layer (three-tier) ---
global rawdata          "\$root/data/raw"          // immutable, gitignored
global workingdata      "\$root/data/cleaned"      // analysis-ready
global tempdata         "\$root/data/temp"         // intermediates, gitignored

* --- Output layer ---
global figure           "\$root/paper/figures"     // .pdf / .png
global table            "\$root/paper/tables"      // .tex (bare tabular)

* --- Code layer ---
global scripts          "\$root/scripts/stata"
global ado              "\$root/scripts/stata/ado"
adopath ++ "\$ado"

* --- Stata defaults ---
set more off
set seed 20260505
set type double
set scheme s2color

* --- Confirm load ---
display "✓ _setup.do loaded for $PROJECT_NAME"
EOF
  echo "📝 Wrote scripts/stata/_setup.do"

  # ---------- Step 4: 写 Python 01_setup.py（mirror Stata macros）----------
  cat > scripts/python/01_setup.py << EOF
"""
01_setup.py — Project root configuration for $PROJECT_NAME
Mirrors Stata's _setup.do macro names for cross-language consistency.

Usage in any analysis script:
    from scripts.python.setup import ROOT, RAWDATA, WORKINGDATA, ...
or:
    exec(open('scripts/python/01_setup.py').read())
"""
from pathlib import Path

# --- Project root ---
ROOT         = Path("$PROJECT_PATH")

# --- Data layer (mirrors Stata) ---
RAWDATA      = ROOT / "data" / "raw"
WORKINGDATA  = ROOT / "data" / "cleaned"
TEMPDATA     = ROOT / "data" / "temp"

# --- Output layer ---
FIGURE       = ROOT / "paper" / "figures"
TABLE        = ROOT / "paper" / "tables"

# --- Code layer ---
SCRIPTS      = ROOT / "scripts" / "python"
FUNCTIONS    = SCRIPTS / "functions"

# --- Defaults ---
import polars as pl  # primary; pandas only for small/legacy

# Reproducibility
import random, numpy as np
SEED = 20260505
random.seed(SEED)
np.random.seed(SEED)

if __name__ == "__main__":
    print(f"✓ 01_setup.py loaded for $PROJECT_NAME")
    print(f"  ROOT = {ROOT}")
EOF
  echo "📝 Wrote scripts/python/01_setup.py"

  # ---------- Step 5: 写一个 master 调度 .do（占位）----------
  cat > scripts/stata/00_master.do << EOF
* ============================================================
* 00_master.do — Run the full pipeline end-to-end
* ============================================================
version 18
do "scripts/stata/_setup.do"

* --- Pipeline ---
* do "\$scripts/02_data_preparation.do"
* do "\$scripts/03_descriptive.do"
* do "\$scripts/04_estimation.do"
* do "\$scripts/05_robustness.do"
* do "\$scripts/06_figures.do"
* do "\$scripts/07_tables.do"

display "✓ Pipeline complete"
EOF
  echo "📝 Wrote scripts/stata/00_master.do (skeleton)"

  # ---------- Step 6: 改 CLAUDE.md 头部为项目身份 ----------
  if [ -f CLAUDE.md ]; then
    # 在 CLAUDE.md 顶部插入项目身份块（保留原内容做参考）
    cat > /tmp/_project_header.md << EOF
# CLAUDE.md — $PROJECT_NAME

**Project:** $PROJECT_NAME
**Stage:** Empirical analysis (early)
**Primary language:** $PRIMARY_LANG
**Branch:** main

> Project-level CLAUDE.md. Project-specific facts (variable definitions,
> cluster levels, sample restrictions, decisions log) belong here.
> Global research DNA lives in ~/.claude/CLAUDE.md.

---

## Decisions Log

<!-- Append decisions here, format:
- YYYY-MM-DD: <decision>. Rationale: <why>.
-->

## Project-Specific Facts

<!-- Variable definitions, sample windows, identification choices, etc.
Add as the project evolves. -->

---

## ↓↓↓ LIBRARY-LEVEL DEFAULTS BELOW (inherited from claude-workflow) ↓↓↓

EOF
    cat CLAUDE.md >> /tmp/_project_header.md
    mv /tmp/_project_header.md CLAUDE.md
    echo "📝 Rewrote CLAUDE.md head as project identity"
  fi

  # ---------- Step 7: gitignore 加项目级条目 ----------
  cat >> .gitignore << EOF

# --- Project-level additions ---
data/raw/
data/temp/
paper/main.aux
paper/main.log
paper/main.bbl
paper/main.blg
paper/main.synctex.gz
*.smcl
EOF
  echo "📝 Augmented .gitignore"

  # ---------- Step 8: git init ----------
  if [ "$INIT_GIT" = "yes" ]; then
    git init -b main 2>&1 | tail -1
    git add -A
    git commit -m "Initial project skeleton from claude-workflow library" 2>&1 | tail -3
    echo "🔖 git initialized + initial commit"
  else
    echo "⊘ Skipped git init (--no-git)"
  fi

  # ---------- Done ----------
  echo ""
  echo "✅ Project ready at: $PROJECT_PATH"
  echo ""
  echo "📋 Next steps:"
  echo "   1. Edit ./CLAUDE.md → fill in project-specific facts"
  echo "   2. ln -s /path/to/your/raw_data data/raw"
  echo "   3. Create GitHub repo:"
  echo "      https://github.com/new (Private, no README/license)"
  echo "      git remote add origin git@github.com:hbli3309-web/$PROJECT_NAME.git"
  echo "      git push -u origin main"
  echo "   4. code ."
  echo ""
  echo "💡 Quick start in Claude Code:"
  echo "   /discover interview <your research question>"
}