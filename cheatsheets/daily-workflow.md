# 你的工作流全景图（一图心智模型）

```
┌─────────────────────────────────────────────────────────────┐
│  ~/.claude/CLAUDE.md   ← 你的 DNA（不动，所有项目继承）        │
├─────────────────────────────────────────────────────────────┤
│  ~/claude-workflow/    ← 图书馆（clo-author fork，慢慢演化）  │
│    └── .claude/        ← agents, skills, rules, settings    │
├─────────────────────────────────────────────────────────────┤
│  ~/Research/<项目名>/  ← 真实项目（每篇论文一个文件夹）        │
│    ├── CLAUDE.md       ← 项目记忆（每天往里加 decisions）     │
│    ├── .claude/        ← 从图书馆拷过来的配置                 │
│    ├── data/           ← 软链接到原始数据                     │
│    ├── scripts/stata/  ← 你和 Claude 一起写的 .do 文件        │
│    ├── paper/          ← 论文本体（main.tex + tables/figures）│
│    ├── quality_reports/← Plans + session logs                │
│    └── explorations/   ← 想法实验室                          │
└─────────────────────────────────────────────────────────────┘
```

**心智模型**：图书馆是**模板**，不在里面写论文。每开一篇新论文 → **拷一份**图书馆出来变成项目 → 在项目里干活。

## CLAUDE.md 的加载顺序（每次会话自动）

```
1. ~/.claude/CLAUDE.md         ← 你的 DNA（识别论、工具默认、协作守则）
2. ~/claude-workflow/CLAUDE.md ← 图书馆约定（agents / rules / skills / Stata 默认）
3. <项目根>/CLAUDE.md          ← 项目记忆（变量、cluster、决策日志）
```

后加载的覆盖前者。所以**项目级**的决策写在项目 CLAUDE.md，不写在图书馆里。

---

# Part 1 · 从零开新项目（每篇论文做一次，约 15 分钟）

## Step 1 · 在 GitHub 网页建空 repo（2 分钟）

打开 https://github.com/new

| 字段 | 填什么 |
|---|---|
| Repository name | `paper-<短名>`（如 `paper-trae-amplification`） |
| Description | 一句论文 working title |
| Visibility | **Private** ✅（论文未发表前永远私有） |
| Initialize with README | ❌ 不勾 |
| Add .gitignore | ❌ 不勾 |
| Add license | ❌ 不勾 |

点 Create。**记住 SSH URL** 长这样：
```
git@github.com:hbli3309-web/paper-trae-amplification.git
```

## Step 2 · 在本地建项目（5 分钟）

VSCode 终端：

```bash
# 1. 进 Research 根目录（如果不存在就建）
mkdir -p ~/Research
cd ~/Research

# 2. 用 clo-author 库的克隆作为起点（不是 fork！是把库的内容拷过来）
git clone --depth 1 git@github.com:hbli3309-web/clo-author.git paper-trae-amplification

cd paper-trae-amplification

# 3. 切断跟 clo-author 的 git 关系，连上你刚建的新 repo
rm -rf .git
git init -b main
git remote add origin git@github.com:hbli3309-web/paper-trae-amplification.git

# 4. 建项目骨架（库里没有的项目级目录）
mkdir -p data/{raw,cleaned} scripts/stata explorations quality_reports/{plans,session_logs}

# 5. 软链接原始数据（不进 git，省磁盘）
ln -s ~/Downloads/ai_coding_sql/raw_data data/raw

# 6. 用 VSCode 打开
code .
```

> **关键概念**：`git clone --depth 1 + rm -rf .git` 是"借用文件但切断历史"的标准做法。你的项目 git 历史从今天的 commit 开始干净起步，跟 clo-author 的几百个 commit 不混在一起。

## Step 3 · 写项目级 CLAUDE.md（5 分钟）

VSCode 里打开根目录 `CLAUDE.md`（它是从图书馆拷来的，目前是**图书馆身份**——头部写的是 `Library: claude-workflow ...`）。

**关键操作**：把头部 4 行从"图书馆身份"改成"项目身份"：

```markdown
# CLAUDE.md — Trae Amplification Paper

**Project:** From Adoption to Amplification — AI Coding Tools and Developer Productivity
**Field:** Empirical labor economics (AI / firm internals)
**Stage:** Empirical analysis + early draft
**Branch:** main
```

下面的 Core Principles / Analysis Language Priority / Folder Structure / Quality Thresholds / Skills Quick Reference 都**保留不动**——它们是图书馆给你的财富。

**再做两件事**（30 秒）：

1. 在文件末尾加一节 `## Decisions log`（空着即可），以后每个变量定义、cluster 选择、sample 限制都按 `- YYYY-MM-DD: <决策>. Rationale: <为什么>` 一行追加。
2. 在文件末尾加一节 `## Paper-to-Code Naming Map`（空着即可）。等开始写 `_setup.do` 时把 paper notation → code variable 的映射放进去（这是 coder.md 的硬要求，跳过会被 coder-critic 扣分）。

## Step 4 · 第一次 commit + push（3 分钟）

VSCode 终端：

```bash
git add -A
git commit -m "Initial project skeleton (cloned from claude-workflow library)"
git push -u origin main
```

**完成**——你新项目的"骨架日"结束。明天进入"开始干活日"。

---

# Part 2 · 日常工作流（每天 3 个动作）

## 动作 1 · 开工（每天早上，30 秒）

```bash
cd ~/Research/paper-trae-amplification
code .
```

VSCode 启动后：
- `Cmd+Esc` 启动 Claude Code 面板
- 检查底部状态栏 `Stata: Connected` —— 必须绿色
- 如果不绿：跑 `curl -s http://localhost:4000/health` 看 MCP server 还在不在

## 动作 2 · 跟 Claude 一起干活（每个任务 1 个循环）

这是核心。**所有工作都按这个 5 步循环**：

```
┌─ 1. 你切 Plan mode (Shift+Tab)
│
│  2. 你描述任务（说人话，不写指令）：
│     "请帮我写 04_estimation.do，跑 post_ai × is_junior 
│      的 TWFE 回归，用 dept_h4 cluster"
│
│  3. Claude 给 plan（约 30 秒）
│     ↓
│     你审 plan：
│     - 步骤合理 → Accept (按 1 或 2)
│     - 有问题 → "第 3 步改成 X" → 它重新规划
│
│  4. Claude 执行：写文件、跑 Stata、生成表
│     ↓
│     每个 diff 弹出来 → 你 Accept / Reject
│
└─ 5. 任务完成 → commit
       VSCode 左侧 Source Control 面板：
       写 commit message → 点 ✓ → push
```

**心智模型**：你是 PI，Claude 是 RA。**Plan mode 是 RA 在白板上画方案给你审**，你 OK 了 RA 才动手。

**plan 必须落到磁盘**：批准 plan 之前让 Claude 把它写到 `quality_reports/plans/YYYY-MM-DD_<短名>.md`。这样 `/compact` 之后下个会话还读得到，不会丢。

## 动作 3 · 收工（每天结束，5 分钟）

不要让你一天的改动堆在那里没 commit。**当晚必须做**：

```bash
# 1. 看今天改了啥
git status
```

**先 `/checkpoint` 再 `/compact` 再 commit**——按这个顺序：

- `/checkpoint` 会把今天的工作摘要写进 `SESSION_REPORT.md` + `quality_reports/research_journal.md` + 自动 memory，下次会话能读到真实上下文，而不是被 auto-compact 压缩出来的那种简短摘要。
- 之后再 `/compact` 释放上下文。
- 最后 commit + push。

```bash
# 2. 让 Claude 帮你写 commit message（如果改动多 / 有结构）
```

在 Claude Code 输入：
```
请看 git status 和 git diff，按改动主题分组，建议 1-3 个语义化 commit message
```

Claude 会给你类似：
```
Suggested commits:
1. "Add 04_estimation.do: TWFE main spec with dept_h4 cluster"
   files: scripts/stata/04_estimation.do, scripts/stata/_setup.do
2. "Update CLAUDE.md decisions log: cluster level confirmed at dept_h4"
   files: CLAUDE.md
```

你审 → Approve → Claude 帮你跑 git add / commit。

```bash
# 3. push 上 GitHub
git push
```

---

# Part 3 · 三件你必须每天做的小事

## 小事 A · 重要决策立刻进 Decisions log（30 秒）

**决策的定义**：变量定义、cluster 选择、sample 限制、identification 选择、robustness 删减。

例子：你今天决定 Junior 阈值用 `tenure_at_entry < 2` 而不是 `< 3`。**立刻**告诉 Claude：

```
请在 ./CLAUDE.md 的 Decisions log 追加：
- 2026-05-05: Junior cutoff = tenure_at_entry < 2 (fixed at first AI use). 
  Rationale: 5+/3+ 阈值会把 senior junior 混在一起；2 年是行业 promotion 节点
```

它会写进去，明天新会话自动读到——**永久记忆**。

## 小事 B · 写 plan 之前先 say plan（10 秒）

不要直接说"改一下 X"。**先说**：

```
我现在要做 X。先 plan。
```

这一句话让 Claude 进 plan mode（比 Shift+Tab 切按钮还自然）。

## 小事 C · 重要 prompt 当下保存（30 秒）

如果你某次跟 Claude 沟通的某段 prompt 效果特别好（比如让它检查 identification 的那段），**立刻**：

```
请把上面那段 prompt 存到 ~/claude-workflow/prompts/identification-audit.md，
作为以后复用模板
```

慢慢攒，半年后你有 30 个高质量 prompt 模板，**这就是你私人的 skill 库**——比图书馆的 skill 更贴你工作。

---

# Part 4 · 11 个 skills 什么时候打哪个？

clo-author 给了你 11 个 `/skill`。你**不需要全记得**，记 5 个高频就够：

| 场景 | 打 | 输出 |
|---|---|---|
| **完全新想法**，要从 0 开始 | `/discover interview <topic>` | Claude interview 你 → 帮你形成研究问题 |
| 想法定型，要设计 identification | `/strategize <question>` | 一份 strategy memo（可被 critic 评分） |
| 数据已 ready，要跑分析 | `/analyze <dataset>` | 完整分析 pipeline + 表 + 图 |
| 写论文章节 | `/write <section>` | 草稿章节 + writer-critic 反馈 |
| 投稿前最终检查 | `/submit` | 模拟 desk review + 2 个 referee |

**剩 6 个**（`/review` `/revise` `/talk` `/checkpoint` `/tools` `/new-project`）等你用上了第一个再说。

---

# Part 4.5 · Stata 快速参考（这个 fork 的默认）

## `_setup.do` 骨架（每个项目复制一遍，不用每次重写）

```stata
version 18
clear all
set type double
set more off

* 项目根 = 当前工作目录（00_master.do 进来时已经 cd 到项目根）
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

**约定**：路径全用小写（`$root` / `$rawdata` / `$workingdata` / `$tempdata` / `$figure` / `$table`），可调常量保留大写（`$SEED` / `$N_BOOT`）。所有 `.do` 不再 `cd`，路径全走这些 macro。这是 INV-16 的硬要求。

## 表格导出（`esttab`，bare tabular，能直接 `\input{}` 进 main.tex）

```stata
eststo clear
eststo m1: reghdfe outcome treatment $controls, absorb(unit_id year) cluster(state_id)

esttab m1 using "$table/main_results.tex",              ///
    replace booktabs fragment                            ///
    b(3) se(3) star(* 0.10 ** 0.05 *** 0.01)             ///
    keep(treatment) coeflabels(treatment "Treatment")    ///
    stats(N r2_a, fmt(0 3) labels("Observations" "Adj. R$^2$")) ///
    nomtitle nonotes nogaps
```

投 AER 系？去掉 `star(...)`，换 `cells("b ci")`（INV-4：AER 不打星号，给置信区间）。

`outreg2` 也支持，但 `booktabs` 要后处理；优先 `esttab`。

## 检查 stata-mcp 还活着

```bash
curl -s http://localhost:4000/health   # 期望返回 ok
```

不响应 → 重启 VSCode 通常就好。`settings.json` 的允许位 `mcp__stata-mcp__*`（带连字符）是这次 fork 改造时加的，缺它会反复弹权限框。

---

# Part 5 · 故障排查 cheatsheet

| 症状 | 先做 |
|---|---|
| Claude 一直要 plan，不执行 | 按 `Shift+Tab` 切到 Auto edits |
| `Stata: Connected` 变灰 | 终端 `curl -s http://localhost:4000/health`；不响应 → 重启 VSCode |
| 命令行 `git push` 提示 HTTPS 凭据 | `git remote -v` 看 URL；`https://` 开头的话改成 `git@github.com:...` |
| 改了 CLAUDE.md 但 Claude 没反应 | 新会话才生效；当前会话发"重新读 @CLAUDE.md" |
| 想撤销 Claude 刚做的改动 | `git status` 看哪些动了 → `git checkout <file>` 单文件还原；或者整体 `git reset --hard HEAD` |
| 跑了一半要换思路 | `/clear` 清当前会话（CLAUDE.md 不丢）然后重新提需求 |

---

# Part 6 · 心理建设：什么不该做

| ❌ 别做 | 为什么 |
|---|---|
| 跳过 Plan mode 直接让 Claude 写 | 多文件改完才发现错——成本是 plan 时回退的 50 倍 |
| 一周不 commit 攒一大堆 | 出问题时 git 救不了你 |
| 不用 `@` 提及文件 | Claude 读不到全仓库；它瞎猜质量很差 |
| 让 Claude 直接动 `~/.claude/CLAUDE.md` | 这是你 DNA，应当**手动**编辑 |
| 在图书馆 `~/claude-workflow/` 里写论文 | 库是模板，不是工作区 |
| 用 `--dangerously-skip-permissions` (Bypass) | 第一年别碰；建立信任前每个 diff 都过一眼 |

---

# Part 7 · 一周时间预算（理想态）

| 频率 | 动作 | 时间 |
|---|---|---|
| 每天 | 开工 + 1-2 个任务循环 + 收工 commit | 视任务而定 |
| 每天 | Decisions log 追加（如有决策） | 30 秒 |
| 每周末 | 让 Claude 读全周 session_logs，写 weekly status | 5 分钟 |
| 每周末 | 检查图书馆有什么用得不爽，记到 ~/claude-workflow/MEMORY.md | 10 分钟 |
| 每月 | 在沙盒里更新 clo-author（`git fetch upstream && git merge`） | 15 分钟 |
| 每季 | 把私人 prompts/ 库整理成新的 skill | 1 小时 |

---

# 一句话总结这套工作流

> **图书馆是模板，项目是工作区，CLAUDE.md 是大脑，Plan mode 是刹车，git 是后悔药。**

每天的循环就 5 步：开工 → 切 plan mode 提需求 → 审 plan → 审 diff → commit。**重复一万次，你就是顶级的 AI-augmented 经济学家**。
