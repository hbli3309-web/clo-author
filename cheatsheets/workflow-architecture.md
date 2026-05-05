# 工作流架构 — 正确做法

> 经过 Sant'Anna 文档验证后的工作流架构。
> 核心原则：每个项目是独立的 .claude/ 拷贝；user-level 只管 CLAUDE.md。

---

## 三层架构

```
┌────────────────────────────────────────────────────────────────┐
│  Layer 1 · USER (一次配置，永久生效)                            │
│  ────────────────────────────────────                          │
│  ~/.claude/                                                     │
│    └── CLAUDE.md   你的研究 DNA（方法论、写作偏好、工具默认）    │
│                                                                  │
│  这里没有 skills/agents/rules/settings.json                     │
│  那些都是 project-level 的事                                     │
└────────────────────────────────────────────────────────────────┘
                          ↑
                          │ 全局继承
                          │
┌────────────────────────────────────────────────────────────────┐
│  Layer 2 · LIBRARY (模板源，不当工作区用)                       │
│  ────────────────────────────────────────                       │
│  ~/claude-workflow/   = fork of hugosantanna/clo-author        │
│    ├── CLAUDE.md      库级配置（描述这个 fork 的约定）           │
│    ├── .claude/       agents, skills, rules, references         │
│    ├── templates/     新项目骨架                                │
│    └── cheatsheets/   你的速查（本文件）                         │
│                                                                  │
│  作用：每开新项目，从这里 clone 一份                             │
│  本身不写论文                                                   │
└────────────────────────────────────────────────────────────────┘
                          ↓
                          │ git clone --depth 1
                          │ rm -rf .git
                          │
┌────────────────────────────────────────────────────────────────┐
│  Layer 3 · PROJECT (每篇论文一个，独立完整)                     │
│  ──────────────────────────────────────────                     │
│  ~/Research/paper-X/                                            │
│    ├── CLAUDE.md      项目级 — 项目身份 + decisions log         │
│    ├── .claude/       项目级 — 完整拷贝自图书馆                  │
│    │   ├── agents/, skills/, rules/, references/, hooks/        │
│    │   └── settings.json                                        │
│    ├── data/, scripts/, paper/, explorations/, ...             │
│    └── (其他项目内容)                                            │
└────────────────────────────────────────────────────────────────┘
```

## Claude Code 实际加载顺序（Sant'Anna §4.6.1）

当你 `cd ~/Research/paper-X && code .` 后启动 Claude Code，配置加载顺序：

1. `~/.claude/CLAUDE.md`（user-level，你的 DNA）
2. `~/Research/paper-X/CLAUDE.md`（project-level，项目身份）
3. `~/Research/paper-X/.claude/rules/*.md`（project-level，path-scoped 规则）
4. `~/Research/paper-X/.claude/skills/*/SKILL.md`（project-level，slash commands）
5. `~/Research/paper-X/.claude/agents/*.md`（project-level，agents）
6. `~/Research/paper-X/.claude/settings.json`（project-level，权限 + hooks）

**user-level 规则：只放 CLAUDE.md。** skills/agents/rules **不能** 放 user-level——它们是 path-scoped 的，依赖项目的目录结构。

## 标准项目目录（Sant'Anna 推荐）

```
~/Research/paper-trae-amplification/
├── CLAUDE.md                       项目身份 + decisions log（项目级）
├── .claude/                        Claude Code 配置（拷贝自图书馆）
│   ├── agents/                     20 个 agent
│   ├── skills/                     11 个 slash command
│   ├── rules/                      9 个规则
│   ├── references/                 coding standards (stata/python/r/julia)
│   ├── hooks/                      protect-files.sh 等
│   └── settings.json               permissions + hooks 配置
├── Bibliography_base.bib           共享文献库
├── MEMORY.md                       项目累积学习（[LEARN] tags）
│
├── paper/                          论文本体（main.tex 是真相之源）
│   ├── main.tex
│   ├── sections/                   按章节拆分的 .tex 文件
│   ├── figures/                    生成的图（pdf/png）
│   ├── tables/                     生成的表（bare tabular .tex）
│   ├── talks/                      Beamer 演讲
│   ├── quarto/                     Quarto RevealJS（可选）
│   ├── preambles/                  共享 LaTeX header
│   ├── supplementary/              在线附录
│   └── replication/                投稿用 replication package
│
├── data/                           数据（raw 通常软链，cleaned 进 git）
│   ├── raw/                        原始数据（软链到磁盘外或 gitignored）
│   └── cleaned/                    清洗后可用的数据
│
├── scripts/                        分析代码
│   ├── stata/                      Stata（本 fork 默认语言）
│   │   ├── _setup.do               【关键】定义所有 macro，version
│   │   ├── 00_master.do            主调度，按顺序跑所有
│   │   ├── 02_data_preparation.do  清洗 + 构造面板
│   │   ├── 03_descriptive.do       summary stats / balance
│   │   ├── 04_estimation.do        主回归
│   │   ├── 05_robustness.do        所有稳健性
│   │   ├── 06_figures.do           所有图（graph export）
│   │   ├── 07_tables.do            所有表（esttab → bare tabular）
│   │   └── ado/                    复用的 .ado 程序
│   ├── python/                     Python（数据工程 + 部分图）
│   │   ├── 00_master.py
│   │   ├── 01_setup.py             定义 ROOT/DATA/TEMP/FIG/TABLE 路径
│   │   ├── functions/              一个文件一个函数
│   │   └── ...
│   └── R/                          R（备用）
│
├── explorations/                   研究沙盒（试验性脚本，不进 paper）
├── quality_reports/                
│   ├── plans/                      每个非琐碎任务的 plan
│   ├── session_logs/               每个 session 的 why-not-just-what
│   ├── specs/                      requirement specs（复杂任务用）
│   └── checkpoints/                /checkpoint 输出
└── master_supporting_docs/         参考文献、数据说明书
```

## Stata 路径 macros（在 `scripts/stata/_setup.do`）

按 ~/.claude/CLAUDE.md 第 3 节约定，全部小写：

```stata
* _setup.do — 项目根级配置，所有 .do 在开头 `do _setup.do` 引入
version 18

* 项目根目录 — 用 `c(pwd)` 或绝对路径
global root      "/Users/hongboli/Research/paper-trae-amplification"

* 数据层 — 三段式
global rawdata        "$root/data/raw"          // 原始数据，只读
global workingdata    "$root/data/cleaned"      // 清洗后可用
global tempdata       "$root/data/temp"         // 中间产物（gitignored）

* 输出层
global figure    "$root/paper/figures"          // .pdf/.png
global table     "$root/paper/tables"           // .tex（bare tabular）

* 工程
global scripts   "$root/scripts/stata"
global ado       "$root/scripts/stata/ado"
adopath ++ "$ado"

* Stata 设置
set more off
set seed 20260505
set type double
```

每个 `.do` 文件开头：

```stata
* 04_estimation.do
version 18
do "/Users/hongboli/Research/paper-trae-amplification/scripts/stata/_setup.do"

use "$workingdata/analysis_panel.dta", clear
* ... 实际工作
```

## Python 路径 mirroring（在 `scripts/python/01_setup.py`）

跟 Stata 用同样命名，方便交叉对照：

```python
# 01_setup.py — 项目根级 Python 配置
from pathlib import Path

ROOT         = Path("/Users/hongboli/Research/paper-trae-amplification")
RAWDATA      = ROOT / "data" / "raw"
WORKINGDATA  = ROOT / "data" / "cleaned"
TEMPDATA     = ROOT / "data" / "temp"
FIGURE       = ROOT / "paper" / "figures"
TABLE        = ROOT / "paper" / "tables"

import polars as pl  # 默认 polars，pandas 仅小数据
```

## 新项目标准初始化流程

```bash
# 1. 选个名字
PROJECT="paper-trae-amplification"

# 2. 从图书馆 clone（不是 fork）
cd ~/Research
git clone --depth 1 git@github.com:hbli3309-web/hb-workflow.git $PROJECT
cd $PROJECT

# 3. 切断跟图书馆的 git 关系
rm -rf .git

# 4. 建项目级目录（图书馆没有的）
mkdir -p \
  data/{raw,cleaned,temp} \
  scripts/stata/ado \
  scripts/python/functions \
  scripts/R/functions \
  paper/{sections,figures,tables,talks,preambles,supplementary,replication} \
  explorations \
  quality_reports/{plans,session_logs,specs,checkpoints} \
  master_supporting_docs

# 5. 软链原始数据（不进 git）
ln -s /path/to/your/raw_data data/raw

# 6. 编辑 CLAUDE.md 头部 — 把 library identity 改成项目身份
#    打开 CLAUDE.md，改前几行的 Library/Maintainer/Adaptation 区域
code CLAUDE.md

# 7. 写 _setup.do 和 01_setup.py，用上面的模板，把 $root 改成本项目路径

# 8. git init + GitHub
git init -b main

# 浏览器去 https://github.com/new 建 private repo: paper-trae-amplification

git remote add origin git@github.com:hbli3309-web/$PROJECT.git
git add -A
git commit -m "Initial project skeleton from claude-workflow library"
git push -u origin main

# 9. 用 VSCode 打开
code .
```

完成后：

- `Cmd+Esc` 启动 Claude Code 面板
- 输入 `/` —— 应该看到 11 个 skill（来自 .claude/skills/）
- 输入 `/discover interview <你的研究问题>` 开始

## 工作流图书馆维护

```bash
# 当 hugosantanna/clo-author 有更新时，sync 到本地图书馆
cd ~/claude-workflow
git fetch upstream
git log HEAD..upstream/main --oneline   # 看新 commit
git diff HEAD upstream/main --stat       # 看改动文件

# 决定要不要合
git merge upstream/main                  # 全要
# 或者只合某个文件
git checkout upstream/main -- path/to/file

# push 到自己的 fork
git push
```

但**已经开过的项目不会自动同步**——它们是图书馆的旧 snapshot。要更新某个旧项目用图书馆新版本：

```bash
cd ~/Research/
# 拷贝图书馆新版的 .claude/ 来覆盖（小心！备份你定制过的 agent/rule）
cp -r ~/claude-workflow/.claude .claude.new
# 手动 diff，merge，然后替换
```

clo-author 也提供 `/tools upgrade` skill 自动做这事，未来用上。