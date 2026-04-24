# OMC ARCHITECTURE.md 分析 — 总结与核心要点

## 📌 3 份分析文档的位置

1. **ARCHITECTURE深度分析.md** — 逐行详细分析（主文档）
2. **ARCHITECTURE可视化快速参考.md** — 可视化图表和快速查询

---

## 一、ARCHITECTURE.md 的核心内容 5 大板块

### 板块 1：Agent 系统（19 个）

**组织方式**：4 个车道

```
Build/Analysis Lane (8)     → 完整开发生命周期
Review Lane (2)             → 质量门槛
Domain Lane (9)             → 按需调用的专家
Coordination Lane (1)       → 挑战和审查
```

**关键 Agent**：

| Agent | 模型 | 成本 | 职责 |
|-------|------|------|------|
| explore | Haiku | ⭐ | **第一步**：发现文件 |
| analyst | Opus | ⭐⭐⭐⭐⭐ | 需求分析 |
| executor | Sonnet | ⭐⭐⭐ | 代码实现 |
| critic | Opus | ⭐⭐⭐⭐⭐ | 计划挑战 |
| verifier | Sonnet | ⭐⭐⭐ | 完成验证 |

**工作流程**（必须顺序）：
```
explore → analyst → planner → critic → executor → verifier
  (发现)   (分析)   (排序)   (审查)   (实现)    (确认)
```

### 板块 2：Skills 系统（31 个技能）

**本质**：行为注入（不替换 Agent，而是在其基础上添加能力）

**组合公式**：
```
[Execution Skill] + [0-N Enhancements] + [Optional Guarantee]
```

**6 个核心 Skill**：

1. **autopilot** — 自动 5 阶段管道（需求→计划→实施→测试→提交）
2. **ralph** — 循环直到验证完成
3. **ultrawork** — 最大并行执行
4. **team** — 多 Agent 协作（5 阶段管道）
5. **ccg** — Claude-Codex-Gemini 3 模型协作
6. **ralplan** — 迭代规划（Planner+Architect+Critic 共识）

### 板块 3：Hooks 系统（事件反应）

**工作原理**：
```
Claude Code 生命周期事件 → Hook 脚本 → OMC 逻辑 → 结果注入
```

**11 个生命周期事件**：

| 事件 | 用途 |
|------|------|
| UserPromptSubmit | 魔法关键词检测 |
| PreCompact | 保存关键信息到 notepad |
| Stop | 持续模式检查、代码简化 |
| SubagentStart/Stop | Agent 追踪 |

**关键 Hook**：
- **keyword-detector** — 检测 "autopilot", "ralph", "ultrawork" 等
- **persistent-mode** — ralph/ultrawork 模式下不停止
- **pre-compact** — 上下文压缩前保存信息

### 板块 4：State 管理（跨上下文重置）

**核心问题**：上下文压缩会重置窗口，工作如何继续？

**解决方案**：将进度和知识保存到 `.omc/` 目录

**两层存储**：

```
Control Plane (快速访问)
├─ .omc/state/           → 操作状态
└─ .omc/notepad.md       → 压缩后恢复

Data Plane (持久成物)
├─ .omc/plans/           → 执行计划
├─ .omc/notepads/        → 知识捕获
└─ .omc/project-memory.json  → 全局知识
```

**三种记忆**：

| 记忆 | 位置 | 生命周期 |
|------|------|---------|
| Notepad | .omc/notepad.md | 压缩后恢复 |
| Project Memory | .omc/project-memory.json | 跨会话（永久） |
| Boulder State | .omc/state/ | 当前会话 |

### 板块 5：验证协议

**7 个检查项**：
- BUILD：编译通过
- TEST：测试通过
- LINT：无错误
- FUNCTIONALITY：功能按预期
- ARCHITECT：Opus 级审查
- TODO：任务完成
- ERROR_FREE：无未解决错误

---

## 二、OMC 架构的 3 个"黑洞"

### 黑洞 1：explore 为什么优先？

**答案**：信息不对称 → 信息采集优先

```
没有信息 → 盲目估计 → 低效失败
↓
有信息 → 精准决策 → 高效成功

explore 的作用：消除信息不对称
```

（详见 `Explore优先使用的决策依据分析.md`）

### 黑洞 2：为什么有 4 个车道？

**答案**：职责清晰，避免重叠

```
Build/Analysis   → 全生命周期（发现→实施→验证）
Review          → 质量检查（安全、代码）
Domain          → 专项需求（测试、文档、UI 等）
Coordination    → 计划审查（critic 挑战计划）
```

### 黑洞 3：为什么有 notepad 和 project-memory？

**答案**：跨越上下文重置的连续性

```
Notepad          → 短期（7 天）：当前任务上下文
Project Memory   → 长期（永久）：项目规则、决策
```

---

## 三、与之前分析文档的联系

```
OMC原理和作用分析.md (宏观)
    ↓
OMC ARCHITECTURE.md (架构设计)
    ↓
Explore优先使用的决策依据分析.md (具体决策)
    ↓
ARCHITECTURE深度分析.md (源码级详解)
```

### 信息层级

```
Level 1：什么是 OMC？
        → OMC原理和作用分析.md

Level 2：OMC 怎样组织？
        → ARCHITECTURE.md

Level 3：为什么 explore 优先？
        → Explore优先使用的决策依据分析.md

Level 4：每个部分的细节？
        → ARCHITECTURE深度分析.md
```

---

## 四、ARCHITECTURE.md 的实用价值

### 对不同角色的价值

| 角色 | 收获 |
|------|------|
| **架构师** | 理解系统设计哲学和职责分离 |
| **开发者** | 知道如何扩展（新 Agent、Skill、Hook） |
| **贡献者** | 理解设计原则，确保一致性 |
| **使用者** | 理解系统如何工作，更好地利用 OMC |

### 关键决策表

**选择 Agent 的依据**：第 140-154 行

```
任务类型              Agent         模型
───────────────────────────────────────
代码查询             explore      Haiku
需求分析            analyst      Opus
计划创建            planner      Opus
计划审查            critic       Opus
代码实现            executor     Sonnet/Opus
```

**选择 Skill 的依据**：第 227-268 行

```
需求                    Skill
─────────────────────────────────
完整自动化             autopilot
循环直到完成           ralph
最大并行               ultrawork
多 Agent 协作          team
3 模型协作            ccg
共识规划              ralplan
```

**成本优化**：第 101-114 行

```
Haiku：快速查询、简单任务
Sonnet：代码实现、调试、测试
Opus：架构、战略分析、审查
```

---

## 五、ARCHITECTURE.md 中的 3 个重要图表

### 图 1：完整执行管道（第 9-36 行）

```
User Input
    ↓
Skill Detection（通过 CLAUDE.md Auto-Routing）
    ↓
Task Type Analysis
    ↓
Skills Matched
    ↓
SKILL ACTIVATED + Parallel agents
```

**含义**：从需求自动推导执行方案

### 图 2：技能分层（第 180-203 行）

```
GUARANTEE LAYER：ralph（不停止直到完成）
ENHANCEMENT LAYER：ultrawork, git-master（增强）
EXECUTION LAYER：default（基础执行）
```

**含义**：Skills 可自由组合，不是必选

### 图 3：生命周期事件→Hook（第 327-343 行）

```
11 个 Claude Code 事件
  ↓
Hook 脚本反应
  ↓
OMC 逻辑执行
  ↓
结果注入回 Claude
```

**含义**：与 Claude Code 的无缝集成

---

## 六、从 ARCHITECTURE.md 出发的深入问题

### 问题 1：为什么是 4 个车道而不是其他数字？

**答案**：职责完全性 + 职责无重叠

```
需要覆盖：发现、分析、规划、审查、实施、验证
建立几个分组：
  Build/Analysis → 核心工作流
  Review → 质量检查
  Domain → 专项任务
  Coordination → 计划挑战
```

### 问题 2：为什么 execute 默认 sonnet 而不是 opus？

**答案**：成本和质量平衡

```
Sonnet 代码实现能力强，成本是 Opus 的 1/3
只在"复杂多文件"时升级到 Opus
```

### 问题 3：为什么需要 Notepad 还需要 Project Memory？

**答案**：时间维度不同

```
Notepad（7 天）     → 当前任务的短期记忆
Project Memory（永久） → 项目的长期记忆

示例：
  Notepad：  "当前计划是修改 auth.ts，已找到问题"
  Project Memory："API 认证采用 JWT，配置在 env"
```

---

## 七、ARCHITECTURE.md 中的 5 个设计原则

### 原则 1：职责明确

```
explore ≠ analyst ≠ planner ≠ executor ≠ verifier
  每个 Agent 只有一个明确职责
```

### 原则 2：自动化优先

```
不让用户选择 Agent
而是从任务类型自动推导

"实施功能" → 自动选 executor
"分析需求" → 自动选 analyst
```

### 原则 3：成本敏感

```
不盲目用 Opus
  Haiku → 快速查询
  Sonnet → 标准代码工作
  Opus → 只在需要时
```

### 原则 4：可恢复性

```
即使上下文重置，也能从 .omc/ 恢复
  notepad → 短期恢复
  project-memory → 长期恢复
```

### 原则 5：可组合性

```
Skills 不是"或"关系，而是"加"关系
default + ultrawork + git-master
  每个 Skill 添加一种能力
```

---

## 八、快速查询指南

### 想要了解...，看第几行？

| 想要了解 | 行号 |
|---------|------|
| Agent 系统总览 | 48-100 |
| 19 个 Agent 详表 | 54-99 |
| 模型路由策略 | 101-114 |
| Agent 选择指南 | 140-154 |
| Agent 职责边界 | 163-170 |
| Skills 概述 | 174-210 |
| 6 大核心 Skills | 227-268 |
| 魔法关键词参考 | 289-307 |
| Hook 系统 | 320-425 |
| State 管理 | 427-593 |
| 验证协议 | 595-610 |

---

## 九、ARCHITECTURE.md 的位置在文档体系中

```
oh-my-claudecode/docs/
├─ ARCHITECTURE.md (本文件)
│  └─ 系统设计的完整蓝图
│
├─ REFERENCE.md
│  └─ 完整 API 参考（更详细）
│
├─ FEATURES.md
│  └─ 内部 API 文档（技术细节）
│
└─ README.md
   └─ 用户指南（如何使用）
```

**阅读顺序**：
1. README.md — 快速开始
2. ARCHITECTURE.md — 理解设计（本文）
3. REFERENCE.md — 查询 API
4. FEATURES.md — 理解实现

---

## 十、总结

**ARCHITECTURE.md 是 OMC 的蓝图**，告诉你：

1. **系统怎样组织**：4 个车道、19 个 Agent、31 个 Skills
2. **怎样决策**：自动推导 Agent、Skill、模型选择
3. **怎样工作**：通过 Hooks 和 State 管理实现无缝编排
4. **怎样恢复**：即使上下文重置也能继续工作
5. **怎样优化**：按复杂度选择模型、按职责选择 Agent

**核心洞察**：

```
OMC = 自动化编排 + 职责明确 + 成本敏感 + 可恢复性

从需求 → 自动推导执行方案 → 执行 → 即使中断也能恢复
```

