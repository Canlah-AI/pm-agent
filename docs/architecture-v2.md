# PM Agent v2: Autonomous Product Iteration Engine

**核心概念**: PM Agent 不是任务分发器。它是一个 meta-programmer —— 像专家用户一样不停操作 Claude Code，迭代产品，直到宇宙无敌。

---

## Core Loop (伪代码)

```python
def run_mission(goal):
    mission = create_mission(goal)

    # Phase 0: 理解 + 规划
    plan = run_claude("分析目标，产出产品愿景 + 质量评分标准", model=sonnet)
    quality_target = extract_target(plan)  # e.g., 85分

    # Phase 1: Build v1
    run_claude("构建初始版本", mode=FULL_SESSION)
    score = run_review()  # 独立的 Claude Code session 评审
    log_iteration(intent="build v1", score=score)

    # Phase 2: 无限迭代直到完成
    stall_counter = 0
    while not is_done(mission):
        # 1. 找到最高优先级的问题
        issues = prioritize_issues(mission)
        if not issues:
            issues = run_deep_review(mission)  # 找不到问题？深度审查
            if not issues:
                mark_complete(mission)
                break

        # 2. 执行修复
        top_issue = issues[0]
        run_claude(plan_fix(top_issue), mode=select_mode(top_issue))

        # 3. 评估（独立 session，无偏见）
        new_score = run_review()
        delta = new_score - previous_score

        # 4. 质量棘轮
        if delta <= 0:
            stall_counter += 1
            record_failed_approach(top_issue)  # 记住什么不管用
            git_revert()  # 回退，不让质量下降
        else:
            stall_counter = 0
            git_commit()  # 锁定进步

        # 5. 定期维护
        if iteration_count % 5 == 0:
            rewrite_memory()      # 重写 MEMORY.md（当前状态，非 changelog）
            send_progress_report() # 通知用户

        # 6. 逃生阀
        if stall_counter >= 3:
            pivot_strategy()      # 完全换个思路
        if total_cost > budget_limit:
            pause_for_approval()  # 暂停问用户
```

---

## 状态模型 (SQLite)

```sql
-- 顶层目标
CREATE TABLE missions (
  id TEXT PRIMARY KEY, goal TEXT, status TEXT,
  quality_target INT, created_at TIMESTAMP
);

-- 每次迭代记录
CREATE TABLE iterations (
  id TEXT PRIMARY KEY, mission_id TEXT, sequence_num INT,
  phase TEXT,  -- build|review|fix|test
  intent TEXT, outcome TEXT,
  quality_score INT, quality_delta INT,
  claude_session TEXT, cost_usd REAL,
  created_at TIMESTAMP
);

-- 已知问题（带优先级）
CREATE TABLE issues (
  id TEXT PRIMARY KEY, mission_id TEXT,
  severity TEXT,  -- critical|major|minor|polish
  description TEXT, status TEXT,
  discovered_iteration INT, resolved_iteration INT
);

-- 关键决策（不可推翻）
CREATE TABLE decisions (
  id TEXT PRIMARY KEY, mission_id TEXT,
  question TEXT, chosen_option TEXT, reasoning TEXT
);
```

---

## 质量评分系统 (0-100)

每次 review 产出结构化 JSON：

| 维度 | 分值 | 评估内容 |
|------|------|---------|
| Functionality | 0-25 | 能用吗？边界情况？|
| Code Quality | 0-25 | 干净、可维护、符合规范？|
| UX/Design | 0-25 | 好看？直觉？响应式？|
| Completeness | 0-25 | 需求都满足了？没有 stub？|

**收敛检测：**
- 连续 3 轮 delta <= 1 → 收益递减，升级或完成
- 分数交替 +/- → 在撤销自己的工作 → 换思路
- score >= target → 完成
- score >= 90 且无 critical/major → 完成

---

## Claude Code 使用决策树

```
需要做什么？
├─ 理解代码     → claude -p "..." --model haiku --max-turns 5
├─ 规划功能     → claude -p "..." --model sonnet --output-format json
├─ 写代码（标准）→ claude -p "..." --model sonnet --max-turns 30
├─ 写代码（关键）→ claude -p "..." --model opus --max-turns 20
├─ 评审质量     → claude -p "..." --model sonnet --output-format json (新 session!)
├─ 修 bug       → claude -p "..." --model sonnet --max-turns 15
├─ 继续上一步   → claude -c -r session-id
└─ 并行任务     → --agents '{...}' (如：同时修 CSS + 写测试)
```

**Session 管理：**
- 新 iteration 新焦点 → 新 session (`-p`)
- 上一轮的后续 → 继续 (`-c`)
- 每个新 session 注入: MEMORY.md + artifact 列表 + 当前 issue + 失败记录

---

## 关键设计决策

| # | 决策 | 原因 |
|---|------|------|
| 1 | **外层循环串行** | 并行迭代会产生冲突，质量评分无意义 |
| 2 | **评审用独立 session** | 没有沉没成本偏见，评估产品 as-is |
| 3 | **MEMORY.md 重写而非追加** | 避免无限增长，只保留"当前重要的" |
| 4 | **每次成功迭代 git commit** | 可回滚，可 diff，可 revert |
| 5 | **预算按用途分配** | 60% build/fix, 25% review, 15% plan（防止跳过评审）|

---

## 用户通信

| 触发 | 类型 | 内容 |
|------|------|------|
| 每 5 轮 | 进度 | 分数趋势 + 做了什么 + 下一步 |
| 分数过 50/70/85 | 里程碑 | "v1 可用 / v2 打磨完 / 接近完美" |
| 卡住 3+ 轮 | 通知 | "在 X 问题上卡住了，换思路中" |
| 预算 > 80% | 紧急 | "暂停，需要你批准继续" |
| 任务完成 | 总结 | 最终分数 + 迭代数 + 总花费 + 关键决策 |

**偏向自治**: 不问许可，只在以下情况暂停：
- 两个同等有效的架构方案
- 需求模糊且解释方式有影响
- 预算超限
- 连续 5+ 轮卡在同一问题

---

## 文件结构 (~1000 行 Python)

```
pm-agent/
├── SKILL.md              # OpenClaw skill 定义
├── pm_engine.py          # 核心循环 + 编排 (~400 行)
├── claude_runner.py      # Claude Code CLI wrapper (~200 行)
├── quality.py            # 评分 + review prompt (~150 行)
├── state.py              # SQLite + MEMORY.md (~150 行)
├── comms.py              # 用户通知 (~100 行)
├── prompts/
│   ├── planner.md        # 初始规划 prompt
│   ├── builder.md        # 代码生成 prompt
│   ├── reviewer.md       # 质量评审 prompt（含评分标准）
│   └── fixer.md          # 修复 prompt
└── schema.sql            # SQLite 建表
```

---

## 示例运行

用户: "帮 CanMarket 做一个宇宙无敌的 landing page"

```
#0  [PLAN]    分析目标，产出愿景 + 评分标准    Score: -    Cost: $0.12
#1  [BUILD]   v1: hero + features + CTA        Score: 42   Cost: $0.35
#2  [FIX]     品牌适配 (Inter, #2563EB)        Score: 58   Cost: $0.28
#3  [FIX]     移动端响应式修复                   Score: 65   Cost: $0.22
#4  [FIX]     动画 + 过渡效果                   Score: 71   Cost: $0.25
#5  [REVIEW]  深度审查：3 major, 5 minor       Score: 71   Cost: $0.18
    → 通知用户: "71/100, 5轮, $1.22"
#6  [FIX]     无障碍：对比度 + aria              Score: 76   Cost: $0.20
#7  [FIX]     性能：懒加载 + CLS                Score: 80   Cost: $0.22
#8  [FIX]     文案打磨 + social proof           Score: 83   Cost: $0.25
#9  [FIX]     SEO + OG image                   Score: 85   Cost: $0.18
#10 [REVIEW]  0 critical, 0 major, 2 polish    Score: 85   Cost: $0.15
    → 通知用户: "85/100 达标！10轮, 47分钟, $2.25. 继续打磨还是发布？"
```

---

## 这个东西 NOT 是什么

- **不是任务路由器** — 它不分发任务，它就是那个脑子
- **不是聊天机器人** — 它不问你下一步做什么，它自己决定
- **不是一次性生成器** — 它不生成就走，它迭代到完美
- **不是 pipeline** — 迭代顺序是动态的，响应产品实际需要

**它是：给一个目标，4 小时后回来收货的那个 senior developer。**
