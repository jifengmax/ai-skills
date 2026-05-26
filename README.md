# AI Skills

🤖 AI Agent 技能仓库 | AI Agent Skills Collection

一个由 **中国人开发** 的 Hermes Agent 技能集合，通过子代理委派（Subagent Delegation）技术，让你的 AI Agent 具备项目经理级的多任务协作能力。

A Hermes Agent skills collection **developed by Chinese developers**, leveraging Subagent Delegation technology to give your AI Agent project-manager-level multi-task collaboration capabilities.

---

## 🚀 核心技能 | Core Skill

### [task-delegation-orchestrator](software-development/task-delegation-orchestrator/SKILL.md)
**任务委派编排师 | Task Delegation Orchestrator**

> 🧑‍💼 **就像设置了一个项目经理** — 你告诉他需求，他会对项目或任务进行分解，按专业领域并行派发子代理执行，逐项审核验收，有问题的打回重做，最终交付严谨结果。
>
> **Like having a project manager** — tell it the requirements, it decomposes the task by domain, dispatches subagents in parallel, reviews each deliverable, reworks failures, and delivers a solid result.

#### ✨ 核心技术 | Core Technologies

- **🧩 Harness 技术 | Harness Architecture** — 编排引擎统一调度多个子代理，确保任务有序执行
- **👥 子代理委派 | Subagent Delegation** — 支持多子代理组成团队并行工作，效果出奇的好
- **🎯 高命中率 | High Accuracy** — 每个子代理专注单一子任务，减少干扰和错误
- **💰 极致 Token 节省 | Extreme Token Efficiency** — 每个子代理只读取自己的任务上下文，**不必完整读取历史对话**，大幅减少 Token 消耗
- **⚡ 并行加速 | Parallel Speedup** — 多个子代理同时工作，速度与效率倍增

#### 📋 工作流程 | Workflow

```
用户 → 项目经理(AI) → 任务分解 → 并行派发子代理 → 逐项审核 → ✅/❌ → 返工/交付
```

#### 🔄 使用方式 | How It Works

> **💡 安装此 Skill 后，每次你派发任务时，AI 会主动问一句：**
> **"是否启用子代理多 Agent 执行模式？"**
> - ✅ 启用 → 调用此 Skill，拆解任务、并行执行、审核返工
> - ❌ 不启用 → 普通模式直接完成（简单任务无需此模式）

---

## 📦 安装方法 | Installation

```bash
# 直接安装本 Skill
hermes skills install https://raw.githubusercontent.com/jifengmax/ai-skills/main/software-development/task-delegation-orchestrator/SKILL.md

# 安装后即可使用
```

---

## 🧪 适用场景 | When to Use

| 场景 | 推荐 | 说明 |
|------|------|------|
| 💻 复杂功能开发 | ✅ 启用 | 前后端分离并行开发 |
| 🐛 复杂 Bug 排查 | ✅ 启用 | 多维度定位 Root Cause |
| 🔧 系统重构 | ✅ 启用 | 多模块分拆并行修改 |
| 📝 简单查询 | ❌ 普通模式 | 直接回答即可 |
| ✏️ 小修小改 | ❌ 普通模式 | 没必要派多个代理 |

---

## 👤 开发者 | Developer

**jifengmax** — 中国开发者 | Chinese Developer

---

## 📄 许可证 | License

MIT
