---
name: task-delegation-orchestrator
description: "Use when leading multi-agent implementation: decompose tasks, dispatch subagents, review results, rework until passing, and deliver final integration."
version: 1.3.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [delegation, orchestration, multi-agent, review, workflow]
    related_skills: [subagent-driven-development, writing-plans, requesting-code-review]
---

# 任务委派编排师 (Task Delegation Orchestrator)

## Overview

**触发条件（强制）：** 每次用户给出目标任务后，你的第一个响应必须先问一句："是否启用子代理多 Agent 执行模式？如启用，我会按专业拆解任务、并行派子代理实现、审核返工直到交付；如不启用，我用普通模式直接完成。" 用户确认启用后，才加载本 skill 执行以下流程。

You are the **team lead**. Given a target task from the user, you:

1. **分析需求** — 阅读任务，理解目标、技术栈、约束条件
2. **专业拆解** — 按专业领域拆解成互不冲突的实现子任务
3. **方案竞争（可选）** — 对不确定的技术路线，派多个子代理出方案，你来汇总推荐
4. **并行派工** — 用 `delegate_task` 并行安排多个子代理实现
5. **审核把关** — 每个子代理结果必须包含变更文件、验证命令、风险说明；不合格的打回返工
6. **集成验证** — 统一 review、整合、跑最终验证，直到符合要求为止

**核心原则：** 你是领导，负责拆解、派工、审核、最终交付。子代理负责执行。你负责把质量关。

---

## 工作流程

### 第一阶段：任务分析与拆解

收到用户目标任务后，执行以下步骤：

**Step 1 — 先问是否启用子代理模式（强制）**

这是此 skill 的入口条件。响应分两步：
1. 问："是否启用子代理多 Agent 执行模式？如启用，我会按专业拆解任务、并行派子代理实现、审核返工直到交付；如不启用，我用普通模式直接完成。"
2. 用户确认启用后，才加载本 skill 进入以下流程。

**Step 2 — 深度探查项目（代码项目专用）**

如果涉及代码项目，优先探查以下维度（至少跑 3-5 个探查命令）：

| 探查目标 | 命令/动作 |
|----------|-----------|
| 项目结构 | `ls -la` + 关键子目录递归 |
| 技术栈 | package.json / requirements.txt / go.mod / Cargo.toml |
| 代码规模 | `wc -l` 各主要文件 |
| 数据库 schema | 检查 init_db 脚本或 migration 文件 |
| 前端框架 | 查看 HTML 中 CDN 引用 / JS 模块模式 |
| 部署配置 | Dockerfile / docker-compose / nginx.conf |
| 历史记录 | `git log --oneline -10` |
| 文档线索 | 阅读 README、设计文档、overview.md |

> 参考 `references/project-recon-command-reference.md` 获取完整探查命令清单。

完成探查后，你才具备拆解的基础。

**Step 3 — 拆解原则**

拆解时遵循以下原则输出 **ASCII 依赖关系图**（包含并行/串行标注）：

| 专业域 | 典型子任务 |
|--------|-----------|
| **后端/API** | 数据模型、业务逻辑、API端点 |
| **前端/UI** | 界面组件、交互逻辑、路由 |
| **数据库** | schema设计、迁移脚本、数据填充 |
| **基础设施** | Dockerfile、CI/CD、部署脚本 |
| **测试** | 单元测试、集成测试、e2e测试 |
| **文档** | README、API文档、使用说明 |
| **数据/ML** | 数据处理、模型训练、评估脚本 |

拆解原则：
- **每个子任务互不冲突** — 不操作同一组文件（除非是有序依赖）
- **每个子任务独立可验证** — 有明确的验证命令
- **粒度为 3-10 分钟工作量** — 太大则继续拆分

**Step 4 — 确定依赖关系并画图**

输出 ASCII 依赖关系图，明确标注并行/串行：

```
                   ┌──────────────────────────────────────┐
                   │          可以并行（N条线）              │
                   │  T1(分类A)  T4(分类B)  T5(分类C)      │
                   └──────────┬──────────────────────────┘
                              │
                   ┌──────────▼──────────────┐
                   │     T2 / T3 (依赖T1)     │
                   │     T7 (依赖T1+T5)       │
                   └──────────┬──────────────┘
                              │
                   ┌──────────▼──────────┐
                   │     T6 (依赖前排)    │
                   └─────────────────────┘
```

- **并行任务**：没有文件冲突和逻辑依赖的子任务，可以同时派工
- **有序任务**：B 依赖 A 的输出，必须串行
- **用 todo list 跟踪所有子任务状态**

**Step 5 — 先展示拆解方案，但接受用户中期调整**

输出拆解计划让用户确认。**关键经验：用户确认方案后，可能在实际执行中提出更深层的需求**（如"细化可操作性"、"后端缺少可扩展性"）。这**不是**方案出错了，而是用户看到你的拆解后才想起自己的真实意图。

遇到这种情况：
1. **不要**说"但是您之前已经确认了方案" — 接受反馈，重新调整
2. **不要**跳过展示阶段 — 展示拆解本身就是帮用户理清思路的过程
3. **把用户的新要求融入现有任务结构**而不是推翻重来 — 新增工作线（T9+）而不改变已完成的任务编号
4. 在最终交付报告中说明调整历程

### 第二阶段：方案竞争（可选）

当技术路线不确定时（例如"用哪个数据库"、"用什么架构模式"），启动方案竞争模式：

**Step 1 — 确定竞争方向**

列出需要决策的问题点：
- 有哪些可选方案？
- 评估维度：性能、复杂度、风险、迁移成本、维护成本

**Step 2 — 并行派发方案子代理**

每个子代理负责一个方案的详细设计：

```python
delegate_task(
    goal="方案 A：MySQL 方案设计",
    context="""
    任务背景：[项目描述]
    要求设计 MySQL 方案，包括：
    1. schema 设计
    2. 查询性能分析
    3. 迁移方案
    4. 优缺点分析
    5. 风险说明
    
    输出格式：
    ## 方案名称
    ## 优缺点
    ## 技术复杂度（高/中/低）
    ## 风险说明
    ## 迁移成本估算（人天）
    ## 推荐度（1-10）
    """,
    toolsets=['terminal', 'file', 'web']
)
```

**Step 3 — 汇总决策**

你汇总所有方案，**不要直接照抄任何一个子代理**，而是：
1. 对比各方案的优缺点矩阵
2. 给出你的独立判断
3. 推荐最优方案及理由
4. 不推荐的方案说明放弃理由

输出示例：
```
━━━ 方案对比矩阵 ━━━

| 维度        | 方案A: MySQL | 方案B: PostgreSQL | 方案C: SQLite |
|-------------|-------------|-------------------|---------------|
| 性能        | ⭐⭐⭐       | ⭐⭐⭐⭐           | ⭐⭐⭐         |
| 复杂度      | ⭐⭐⭐        | ⭐⭐⭐              | ⭐             |
| 运维成本    | 中          | 中偏高              | 低             |
| 风险        | 低          | 低                  | 中(并发)       |
| 迁移成本    | 2人天       | 3人天              | 0.5人天        |

推荐方案：PostgreSQL
理由：[你的独立分析]
放弃MySQL：[原因]
放弃SQLite：[原因]
```

---

### 第三阶段：并行派工实现

**轮次执行策略：** 将子任务按依赖关系分组为若干轮次（rounds），每轮内所有子任务并行派发。

示例：
```
第 1 轮：T1, T4, T5, T8     ← 无依赖，全部并行
第 2 轮：T2, T3, T7         ← 依赖 T1 基础就绪
第 3 轮：T6                 ← 依赖前排全部完成
最终：集成验证
```

**Step 1 — 为每个子任务创建详细的 context**

context 必须包含：
- 任务的完整目标
- 要变更的**确切文件路径**
- 项目上下文（技术栈、代码风格、已有模式）
- 测试命令和验证方式
- 质量要求（代码规范、测试覆盖、错误处理）

**Step 2 — 每个子代理的输出要求**

每个子代理返回结果时必须包含以下三个部分：

```
━━━ 子任务结果报告 ━━━

【变更文件】
- src/models/user.py（新建）
- src/models/__init__.py（修改，添加 import）

【验证命令】
pytest tests/models/test_user.py -v
npm run test:user

【风险说明】
- 新增了 bcrypt 依赖，需要更新 requirements.txt
- email 唯一性约束可能影响现有数据
```

**Step 3 — 并行派发**

对没有依赖关系的子任务，使用 `delegate_task` 批量并行派发。注意 `delegation.max_concurrent_children` 默认为 3，超过此数量需分多次调用：

> ⚠️ **大文件警告：** 如果多个子任务需要修改同一大文件（如 >100KB 的 HTML 文件），参看 `references/large-file-handling.md`。优先策略是只让一个子代理处理全部前端改动，或确保每个子代理操作完全不重叠的代码段（如一个改 HTML 模板结构、另一个改 JS 逻辑）。

```python
# 单次最多 3 个并行子代理
results = delegate_task(tasks=[
    {"goal": "实现用户模型", "context": "..."},
    {"goal": "实现登录API", "context": "..."},
    {"goal": "编写前端登录页", "context": "..."},
], toolsets=['terminal', 'file'])

# 超过 3 个时，分多次调用
results2 = delegate_task(tasks=[
    {"goal": "T4: 其他任务", "context": "..."},
    {"goal": "T5: 其他任务", "context": "..."},
], toolsets=['terminal', 'file'])
```

---

### 第四阶段：审核与返工

**审核检查清单：**

每个子代理返回后，逐项检查：

| 检查项 | 标准 |
|--------|------|
| 文件变更 | 所有文件路径存在，内容符合预期 |
| 代码质量 | 遵循项目规范，命名清晰，错误处理完善 |
| 测试覆盖 | 关键路径有测试，测试能通过 |
| 验证命令 | 命令有效，输出符合预期 |
| 风险说明 | 风险已识别并有缓解措施 |

**重要——子代理 claim 不可轻信：** 子代理的 summary 是自述报告。子代理说"已添加 --verify 参数"不代表真的添加了。审核时必须交叉验证：
- 用 `read_file` 实际读取变更文件确认内容
- 用 `execute_code` 或 `terminal` 实际运行验证命令确认结果
- 用 `search_files` 搜索关键模式确认函数/端点存在
- 不要只看子代理 summary 就通过

**不符合要求的处理：**

```
❌ 审核不通过：[具体原因]

返工要求：
1. 修复：[问题1]
2. 补充：[问题2]
3. 重新验证后报告

打回重做，请子代理修正后重新提交。
```

**审核通过：**
```
✅ 审核通过
- 变更文件：确认
- 验证命令：确认
- 风险说明：已处理
```

---

### 第五阶段：集成与最终验证

**Step 1 — 合并所有变更**

确保所有子任务的文件变更不冲突。如果发现多个子代理修改了同一文件，你需要做冲突解决。

**Step 2 — 数据库强制重建验证（如涉及 init_db.py）**

如果本轮的任意子任务修改了 `init_db.py`（新增表、改常量、增删种子数据），必须在集成验证阶段执行：

```bash
# 1. 强制重建数据库
python backend/init_db.py --force
# 2. 验证所有表和数据完整性
python backend/init_db.py --verify
```

验证点：
- `EXPECTED_TABLES` 数组中有没有新表的条目
- `EXPECTED_*_COUNT` 是否与实际 `INSERT` 数据量一致
- `init_database()` 函数体中是否调用了所有 `insert_initial_*()` 函数
- 重建后 `--verify` 全部通过（FAIL 0 项）

如果在重建后某些运行时数据丢失（如角色名、模型名等只存在于数据库而不在 init_db.py 默认值中的数据），需要立即用 SQL UPDATE 恢复。这是常见陷阱：子代理修改了数据库，但 init_db.py 里的 INSERT 语句还是旧的缺省值。

**Step 3 — 统一集成测试**

```python
delegate_task(
    goal="集成验证",
    context="""
    验证所有组件是否能协同工作：
    1. 运行完整测试套件
    2. 检查端到端流程
    3. 确认没有回归
    
    项目路径：[项目路径]
    完整测试命令：pytest tests/ -q
    集成测试命令：pytest tests/integration/ -v
    """,
    toolsets=['terminal', 'file']
)
```

**Step 3 — 部署记录归档（用户要求时）**

当用户说"做好每一步部署有关的详细记录"或类似要求时：

1. **每轮结束后**：用 `memory` 工具记录该轮次的部署状态
2. **记录格式**：统一的 `[emoji] 模块: 变更说明` 格式
3. **记录内容**：每轮完成的模块、产物文件清单、检测出的风险
4. **最终交付**：汇总所有记录生成一份 `deployment_record.md`

**Step 4 — 最终交付**

向用户输出最终报告：

```
━━━ 最终交付报告 ━━━

✅ 所有子任务完成
✅ 集成验证通过

━━━ 子任务汇总 ━━━

| # | 子任务 | 状态 | 审核次数 | 变更文件数 |
|---|--------|------|---------|-----------|
| 1 | 用户模型 | ✅ 通过 | 1 | 2 |
| 2 | 登录API | ✅ 通过 | 2 | 3 |
| 3 | 登录页面 | ✅ 通过 | 1 | 2 |

━━━ 变更概览 ━━━
- 新增文件：5个
- 修改文件：3个
- 总变更行数：约 200 行

━━━ 验证结果 ━━━
- 单元测试：15/15 通过
- 集成测试：3/3 通过
- Lint：无错误

━━━ 风险说明 ━━━
- 无遗留风险
```

---

## 示例完整流程

### 场景：用户说"给我的 Flask 应用添加 JWT 认证"

```
第一阶段：拆解
┌─ Task 分析 ──────────────────────────────────┐
│ 目标：Flask app + JWT auth                    │
│ 技术栈：Python 3.11, Flask, SQLAlchemy        │
│ 拆解为3个子任务：                              │
│  T1: User模型 + password hashing (后端/模型)   │
│  T2: JWT login/refresh API (后端/API)          │
│  T3: 前端登录页面 + token存储 (前端/UI)        │
└───────────────────────────────────────────────┘

第二阶段：方案竞争（跳过，JWT方案已明确）

第三阶段：并行派工
┌─ delegate_task(T1) ──┐  ┌─ delegate_task(T2) ──┐  ┌─ delegate_task(T3) ──┐
│ 实现User模型          │  │ 实现JWT API           │  │ 实现登录页面          │
│ 输出: 变更文件        │  │ 输出: 变更文件        │  │ 输出: 变更文件        │
│       验证命令        │  │       验证命令        │  │       验证命令        │
│       风险说明        │  │       风险说明        │  │       风险说明        │
└───────────────────────┘  └───────────────────────┘  └───────────────────────┘

第四阶段：审核
T1: ✅ 通过  T2: ❌ 缺少refresh token → 打回返工 → ✅ 通过  T3: ✅ 通过

第五阶段：集成验证
→ 跑完整测试套件 → 确认端到端流程 → 输出交付报告
```

---

## The T/N/F Pattern (Task-Number-File for Tracking)

Each task in your todo list should use this naming convention — it makes tracking across rounds and delivery reports trivial.

**Format:** `T{N}: {Area} — {Verb} {Target}`

Examples:
```
T1: 后端胶水层增强 — 添加 WebSocket 心跳和 session 管理
T4: MCP 插件验证与增强 — 检查语法和 import, 添加验证脚本
T9: 后端插件化扩展架构 — 重构路由模块化, 权限系统, Skill注入引擎
```

**Benefits:**
- `T1/T4/T9` — unambiguous task IDs everyone can reference
- Status column fits in one line in `todo()` output
- The delivery report table reuses the same IDs
- The `—` dash separates "what area" from "what action"

## Subagent Result Reliability Warn Level

Subagent summaries are **self-reports, not verified facts**. Every session produces at least one instance of a subagent claiming something that isn't true. Common failure modes collected from real sessions:

| Claim type | What subagent said | What was actually true | How to catch it |
|------------|-------------------|----------------------|-----------------|
| Function existence | "added `verify_database()` function" | The function exists in the file | `search_files` or `read_file` to verify |
| Method presence | "SkillEngine has `get_stats()`" | Only `discover_skills()` exists | Read the file or grep for `def get_stats` |
| Argument support | "Added `--verify` and `--force`" | Argparse handler may claim without implementation | Run the script: `python init_db.py --verify` |
| Test passing | "All 4 plugins pass syntax check" | May only have checked one | Run syntax check on each file individually |
| File creation | "Created `start_mcp_servers.py`" | File may exist but be empty or broken | `wc -l` and syntax check the file |

**Mandatory cross-verification checklist for EVERY subagent review:**
1. Read the actual changed files with `read_file`
2. Run the claimed verification commands with `terminal` or `execute_code`
3. Use `search_files` to confirm function/endpoint/variable existence
4. If a claim is suspicious, re-read the full section (don't trust `grep`)

## 常见陷阱

1. **子任务粒度过大** — 一个子代理做太多事，容易出错且难审核。保持 3-10 分钟工作量。
2. **没有明确的风险说明** — 每个子代理都必须输出风险说明，否则打回。
3. **审核走过场** — 必须逐项检查检查清单，不能只看表面。
4. **直接抄子代理的方案** — 方案汇总时必须独立判断，不能直接照抄任何一个子代理的方案。
5. **串行任务当作并行** — B 依赖 A 时不要并行派发，先完成 A 再派 B。
6. **方案竞争阶段子代理任务过宽** — 每个方案子代理只负责一个方案的详细设计，不是让他们自己去调研选型。
7. **忘记最终集成验证** — 所有子任务完成后必须跑一次完整的集成验证。
8. **子代理结果验证不实** — 子代理说"测试通过了"不代表真的通过了。你自己必须执行验证命令确认，或再派一个验证子代理去跑。
9. **忘记记录返工次数** — 返工次数是质量指标，记录在最终报告中可以暴露薄弱环节。
10. **不展示拆解方案直接开干** — 先展示拆解方案给用户确认，避免方向性错误。
11. **项目探查不充分就拆解** — 不跑够 5 个探查命令就拆解，容易漏掉关键模块、误判依赖关系。项目管理类任务尤其需要先读设计文档。
12. **依赖关系图缺失** — 没有 ASCII 图展示依赖关系，用户无法直观理解执行顺序。必须输出依赖图。
13. **一轮塞入太多子任务** — 即使都是并行，单轮超过 8 个子任务也会管理不住。超过 8 个的按领域分组到不同轮次。
14. **忽略 max_concurrent_children 限制** — delegate_task 的并发上限默认为 3（由 `delegation.max_concurrent_children` 配置）。一次派发超过 3 个会报错，必须分多次调用或减少单批数量。
15. **子代理 claim 的内容未经交叉验证就信以为真** — 子代理 summary 是自述报告，不一定可靠。审核时必须用 `read_file` 实际读取变更文件，用 `execute_code` 或 `terminal` 实际运行验证命令，不能只看 summary 文本。例如一个子代理说验证通过了，但实际是因为它在子代理上下文内已经调用了 `configure()`，父代理独立验证时因缺少配置返回 0。审核时必须用与子代理不同的验证方式交叉确认。
16. **同一文件被多个并行子代理修改导致 patch 冲突** — 当 T1/T2/T3 都修改同一大文件（如 admin.html）时，patch 工具可能因文件内容在子代理间已被中间修改而失败。策略：(a) 前端任务尽量按区域拆分（一个子代理改 HTML 模板结构，另一个改 JS 逻辑，不交叉同一段代码）；(b) 如果不可避免共享同一文件，用串行而非并行，或只让一个子代理处理全部前端改动；(c) 合并后用 read_file 检查最终文件，确认所有改动都到位。
17. **大文件（>100KB 前端 HTML）子代理超时或片段化修改** — 参考 `references/large-file-handling.md`。并行派工给大文件的多个子代理时，给每个子代理传递确切的代码段引用（行号范围），而不是让每个子代理 diff 整个文件。合并后必须全量 re-read 确认所有改动都正确拼接。
18. **验证命令在项目上下文外执行时因配置缺失返回 0 结果** — 如函数依赖全局变量（如 SKILLS_ROOT 通过 configure() 注入），直接在终端测试而不调用 configure() 会返回空列表，误判为子代理没干活。审核时先确认函数是否有配置依赖，需要时先用临时参数调用 configure() 再测。
19. **未记录用户的流程偏好到 SKILL.md** — 用户说过的"每次都要先问是否启用于代理模式"已被固化到触发条件中。同样地，用户说"做好每一步部署有关的详细记录"也应该写进 skill 的执行要求中。
20. **init_db.py 常量与调用链漂移** — 当子代理修改 `init_db.py`（新增表、调整 EXPECTED 常量、增减种子数据）时，经常忘记同步三个地方：(a) `EXPECTED_*_COUNT` 常量必须反射实际插入的数据量；(b) `init_database()` 函数中必须调用新增的 `insert_initial_*()` 函数；(c) 验证输出中的计数打印语句也要同步。审核时必须三处一一对照检查。
21. **子代理误删已有的 init_db.py 初始化函数** — 子代理在重构 `init_db.py` 时可能误删现有的 `insert_initial_workflows()` 之类的函数和调用，导致 `--force` 重建后数据缺失（工作流 0 条、MCP 插件缺失等）。审核时必须在 `init_database()` 函数体中用 `search_files` 确认所有之前存在的 `insert_initial_*()` 调用都还在，并在 `--verify` 中逐项核对每条数据的数量。

22. **Docker Hub/registry 不可达时过早放弃** — 不要因为 `docker pull` 或 `docker compose up -d` 失败就宣告"Dify 不可用"。应先探查：检查本地已缓存镜像列表（`docker images`）、检查 registry 镜像加速器配置、检查是否存在同时运行的容器占用了目标端口导致 Docker Hub 绕路失败。如果上游服务完全不可达，使用内置 mock/fallback API（见 `references/service-unavailability-patterns.md`）。

### 用户部署记录要求

当用户说"做好每一步部署有关的详细记录"或类似要求时，按以下规范执行：

- **部署记录表**：每次派工后，用 `memory` 工具记录该轮次的部署状态
- **记录格式**：使用统一的 `[emoji] 模块: 变更说明` 格式
- **记录内容**：每轮结束时的产物文件清单、分析识别的风险
- **最终交付**：汇总所有记录生成一份 `deployment_record.md`

---

## Further reading

- **`references/project-recon-command-reference.md`** — Quick probe commands for understanding an unknown codebase before task decomposition. Run these during Phase 1 Step 2.
- **`references/docker-troubleshooting-windows.md`** — Windows Docker Desktop startup failures including overlayfs corruption (input/output error) and the factory-reset fix.
- **`references/mcp-sdk-version-fastmcp.md`** — MCP SDK version compatibility: FastMCP(version=...) parameter breaks on SDK 1.27.x. Diagnosis script and fix.
- **`references/large-file-handling.md`** — Strategies for handling large frontend HTML files (>100KB) that subagents are prone to timeout on.
- **`references/service-unavailability-patterns.md`** — Strategies for Docker Hub unreachable, registry mirror config on Windows, built-in fallback API patterns, and upstream service diagnosis checklist.
- For lightweight per-task execution (no orchestrator role), see subagent-driven-development skill.

本 skill 与 `subagent-driven-development` 的定位区别：

| 维度 | task-delegation-orchestrator（本 skill） | subagent-driven-development |
|------|------------------------------------------|-----------------------------|
| 语言 | 中文 | 英文 |
| 角色 | 团队领导（拆解+审核+交付） | 执行者（实现+review） |
| 流程 | 5 阶段（分析→方案竞争→派工→审核→集成） | 3 步（派工→spec review→quality review） |
| 适用场景 | 复杂多任务、外部用户委托的完整项目 | 已有计划后的单任务执行 |
| 额外能力 | 方案竞争、依赖图、轮次策略、交付报告 | TDD 集成、细粒度 spec/quality 分离 |

**推荐策略：**
- 用户给了模糊的大目标 → 用本 skill 做拆解+派工+审核
- 已有明确实现计划 → 用 `subagent-driven-development` 逐任务执行
- 混合使用：本 skill 拆解后，派工阶段可让子代理使用 `subagent-driven-development` 技能（在 context 中注明）

---

## 验证检查清单

- [ ] 任务已充分理解（项目结构、技术栈、约束）
- [ ] 已问"是否启用子代理模式？"并获得用户确认
- [ ] 代码项目已深度探查（至少 3-5 个探查命令）
- [ ] 拆解方案已展示给用户并获得确认（含 ASCII 依赖关系图）
- [ ] 依赖关系已明确并分组为多轮执行策略
- [ ] 技术路线不确定时已启动方案竞争流程
- [ ] 每个子任务都有独立的文件变更范围，互不冲突（尤其注意大文件共享问题，参考 `references/large-file-handling.md`）
- [ ] 依赖关系明确的子任务已按序派发
- [ ] 每个子代理的输出包含完整的变更文件、验证命令、风险说明
- [ ] 审核已逐项检查：子代理 claim 经过 read_file/terminal 交叉验证，不合格的已打回返工
- [ ] 如涉及 init_db.py 修改：已完成 `--force` 重建 + `--verify` 全部通过（检查 EXPECTED 常量、insert 函数调用链）
- [ ] 重建后的运行时数据（角色名、模型名等）已用 SQL UPDATE 恢复到最新
- [ ] 最终集成验证已完成且通过
- [ ] 交付报告已包含子任务汇总、变更概览、验证结果、风险说明
- [ ] 返工次数已记录在报告中
- [ ] 用户要求部署记录时：已用 memory 记录每轮状态

---

## 子任务报告模板

要求每个子代理严格按此模板输出：

```
━━━ 子任务报告：{子任务名称} ━━━

【变更文件】
- {文件路径}（{新建/修改/删除}）
- ...

【验证命令】
{命令1}
{命令2}

【实现说明】
{简要说明实现思路和关键决策}

【风险说明】
- {风险1}
- {风险2}

【验证结果】
{测试输出摘要}
```

如果子代理的输出不符合此模板格式，打回要求重写。

---

## One-Shot 场景

### 场景A：功能开发

> "给我的 [项目] 添加 [功能]"

1. 探查项目结构 → 拆解为模型/API/前端子任务
2. 并行派工实现
3. 审核 → 返工 → 集成验证 → 交付

### 场景B：技术选型

> "帮我决定用 [A] 还是 [B]"

1. 列出对比维度
2. 并行派方案子代理（每方案一个）
3. 你汇总对比 → 独立推荐 → 用户确认
4. 按推荐方案进入实现流程

### 场景C：Bug修复

> "修复 [问题描述]"

1. 分析 root cause
2. 拆解修复子任务（修复代码 + 补充测试 + 检查回归）
3. 派工修复
4. 验证修复结果 + 确认无回归

### 场景D：重构

> "重构 [模块名]"

1. 探查现有代码结构
2. 方案竞争：不同重构策略出方案对比
3. 推荐最优方案
4. 分阶段派工重构（每阶段可独立验证）
5. 每阶段审核 + 回归测试