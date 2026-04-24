# OMC ARCHITECTURE.md 深度分析

## 文件概览

**位置**: `/docs/ARCHITECTURE.md`
**作用**: 阐述 OMC 的四大核心系统架构和运作机制
**目标读者**: 开发者、架构师、贡献者

---

## 一、核心架构图解（619-36）

### 1.1 整体架构流

```
User Input --> Hooks (event detection) --> Skills (behavior injection)
           --> Agents (task execution) --> State (progress tracking)
```

这是 OMC 的**完整执行管道**：

| 阶段 | 组件 | 作用 |
|------|------|------|
| 1 | User Input | 用户输入 |
| 2 | Hooks | **事件检测** — 截获生命周期事件 |
| 3 | Skills | **行为注入** — 激活高级工作流 |
| 4 | Agents | **任务执行** — 19 个专门化 Agent |
| 5 | State | **进度追踪** — 保存状态跨越上下文重置 |

### 1.2 可视化流程（第 9-36 行）

```
User Input: "ultrawork refactor the API"
                ↓
        CLAUDE.md Auto-Routing
                ↓
        Task Type Analysis:
        - Implementation ✓
        - Multi-file ✓
        - Parallel OK ✓
                ↓
        Skills Matched:
        - ultrawork ✓
        - default ✓
        - git-master ✓
                ↓
        SKILL ACTIVATED
        ├─ Parallel agents launched
        └─ Atomic commits enabled
```

**关键观察**：系统从"什么要做"推导出"如何做"，自动选择技能组合

---

## 二、Agent 系统详解（第 48-171 行）

### 2.1 Agent 分类架构

OMC 将 19 个 Agent 分为 4 个**车道（Lane）**，每个车道有不同的职责：

```
┌──────────────────────────────────────────────┐
│ Build/Analysis Lane (8个)                    │
│ 完整开发生命周期                              │
│ explore → analyst → planner → architect →   │
│ debugger → executor → verifier → tracer    │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ Review Lane (2个)                            │
│ 质量门槛                                      │
│ security-reviewer                            │
│ code-reviewer                                │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ Domain Lane (9个)                            │
│ 领域专家（按需调用）                         │
│ test-engineer, designer, writer, qa-tester │
│ scientist, git-master, document-specialist  │
│ code-simplifier                              │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ Coordination Lane (1个)                      │
│ 挑战和审查                                    │
│ critic — Gap analysis expert                │
└──────────────────────────────────────────────┘
```

### 2.2 Agent 详细对照表

#### Build/Analysis Lane（第 54-67 行）

| Agent | 模型 | 成本 | 职责 |
|-------|------|------|------|
| explore | haiku | ⭐ | 代码库发现、文件映射 |
| analyst | opus | ⭐⭐⭐⭐⭐ | 需求分析、隐藏约束发现 |
| planner | opus | ⭐⭐⭐⭐⭐ | 任务排序、执行计划创建 |
| architect | opus | ⭐⭐⭐⭐⭐ | 系统设计、接口定义、权衡分析 |
| debugger | sonnet | ⭐⭐⭐ | 根本原因分析、构建错误解决 |
| executor | sonnet | ⭐⭐⭐ | 代码实现、重构 |
| verifier | sonnet | ⭐⭐⭐ | 完成验证、测试充分性 |
| tracer | sonnet | ⭐⭐⭐ | 证据驱动追踪、竞争假设分析 |

**工作流**（第 157-161 行）：
```
explore → analyst → planner → critic → executor → verifier
(发现)   (分析)   (排序)   (审查)   (实现)    (确认)
```

#### Review Lane（第 69-76 行）

| Agent | 模型 | 职责 |
|-------|------|------|
| security-reviewer | sonnet | 安全漏洞、信任边界、认证/授权 |
| code-reviewer | opus | 综合代码审查、API 契约、向后兼容性 |

**关键点**：review lane 不修改代码，只审查

#### Domain Lane（第 78-91 行）

这些是"按需调用"的专家：

| Agent | 职责 |
|-------|------|
| test-engineer | 测试策略、覆盖率分析 |
| designer | UI/UX 架构、交互设计 |
| writer | 文档、迁移注记 |
| qa-tester | 交互式 CLI/服务运行时验证（tmux） |
| scientist | 数据分析、统计研究 |
| git-master | Git 操作、提交、变基、历史管理 |
| document-specialist | 外部文档、API/SDK 参考查询 |
| code-simplifier | 代码清晰度、简化、可维护性改进 |

#### Coordination Lane（第 93-99 行）

| Agent | 职责 |
|-------|------|
| critic | 计划和设计的间隙分析、多角度审查 |

**重要**：critic 是"挑战者"，计划必须通过 critic 的审查才算有效

### 2.3 模型路由策略（第 101-114 行）

三层模型架构：

```
┌─────────────────────────────────────────┐
│ LOW TIER (Haiku)                        │
│ - 快速查询和简单任务                    │
│ - explore, writer                       │
│ - 成本最低                              │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ MEDIUM TIER (Sonnet)                    │
│ - 代码实现、调试、测试                  │
│ - executor, debugger, test-engineer    │
│ - 成本和质量平衡                        │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ HIGH TIER (Opus)                        │
│ - 架构、战略分析、审查                  │
│ - architect, planner, critic, code-reviewer
│ - 成本最高，质量最高                    │
└─────────────────────────────────────────┘
```

**路由原则**（第 111-114 行）：
- Haiku：快速查询、简单任务
- Sonnet：代码实现、调试、测试
- Opus：架构、战略分析、审查

### 2.4 Agent 选择指南（第 140-154 行）

实用表格，根据任务类型选择 Agent：

| 任务类型 | 推荐 Agent | 模型 |
|---------|-----------|------|
| 快速代码查询 | explore | haiku |
| 功能实现 | executor | sonnet |
| 复杂重构 | executor | opus |
| 简单 Bug 修复 | debugger | sonnet |
| 复杂调试 | architect | opus |
| UI 组件 | designer | sonnet |
| 文档 | writer | haiku |
| 测试策略 | test-engineer | sonnet |
| 安全审查 | security-reviewer | sonnet |
| 代码审查 | code-reviewer | opus |
| 数据分析 | scientist | sonnet |

### 2.5 Agent 职责边界（第 163-170 行）

这是**严格的职责划分**，防止重叠：

| Agent | 做什么 | 不做什么 |
|-------|-------|---------|
| architect | 代码分析、调试、验证 | 需求收集、计划制定 |
| analyst | 查找需求缺口 | 代码分析、计划制定 |
| planner | 创建任务计划 | 需求分析、计划审查 |
| critic | 审查计划质量 | 需求分析、代码分析 |

**原理**：每个 Agent 有明确的"边界"，不越界

---

## 三、Skills 系统详解（第 174-318 行）

### 3.1 Skills 的本质（第 174-210 行）

**Skills 是行为注入**，不是替换 Agent，而是**在 Agent 基础上添加能力**：

```
公式: [Execution Skill] + [0-N Enhancements] + [Optional Guarantee]

例如：
Task: "ultrawork: refactor API with proper commits"
活跃技能：ultrawork + default + git-master
           ↑          ↑          ↑
         增强       执行层    增强
```

### 3.2 技能分层（第 180-203 行）

```
┌──────────────────────────────────────────┐
│ GUARANTEE LAYER (可选)                   │
│ ralph: "直到验证完成才停止"              │
└──────────────────────────────────────────┘
         ↓ (集成到)
┌──────────────────────────────────────────┐
│ ENHANCEMENT LAYER (0-N 个技能)            │
│ ultrawork (并行)                         │
│ git-master (提交)                        │
│ frontend-ui-ux (其他增强)               │
└──────────────────────────────────────────┘
         ↓ (集成到)
┌──────────────────────────────────────────┐
│ EXECUTION LAYER (主技能)                  │
│ default (构建)                           │
│ orchestrate (协调)                       │
│ planner (规划)                           │
└──────────────────────────────────────────┘
```

**关键观察**：技能是可组合的（composable）

### 3.3 核心工作流技能（第 227-268 行）

#### autopilot（自动驾驶）

```
触发：autopilot, build me, I want a

示例：
autopilot build me a REST API with authentication

工作流：5 个阶段完整管道
├─ 理解需求
├─ 创建计划
├─ 实现代码
├─ 测试验证
└─ 提交代码
```

#### ralph（自我反思循环）

```
触发：ralph, don't stop, must complete

示例：
ralph: refactor the authentication module

特点：不停止直到工作被验证完成
流程：
├─ PRD Phase: 需求定义
├─ Progress Phase: 执行
├─ Verification Phase: 验证
└─ 若失败，循环回 PRD Phase
```

#### ultrawork（并行执行）

```
触发：ultrawork, ulw

示例：
ultrawork implement user authentication with OAuth

特点：最大并行度 — 同时启动多个 Agent
```

#### team（多代理协作）

```
语法：/oh-my-claudecode:team N:executor "..."

示例：
/oh-my-claudecode:team 3:executor "implement fullstack todo app"

5 阶段管道：
plan → prd → exec → verify → fix

"3:executor" 表示 3 个 executor 并行
```

#### ccg（Claude-Codex-Gemini）

```
触发：ccg, claude-codex-gemini

特点：同时扇出到 Codex 和 Gemini；Claude 综合结果

示例：
ccg: review this authentication implementation
```

#### ralplan（迭代规划）

```
触发：ralplan

特点：Planner, Architect, Critic 循环直到达成共识

示例：
ralplan this feature
```

### 3.4 技能调用方式（第 211-225 行）

**两种调用方式**：

1. **斜杠命令**
```bash
/oh-my-claudecode:autopilot build me a todo app
/oh-my-claudecode:ralph refactor the auth module
/oh-my-claudecode:team 3:executor "implement fullstack app"
```

2. **魔法关键词**（自动激活）
```bash
autopilot build me a todo app           # 自动激活 autopilot
ralph: refactor the auth module         # 自动激活 ralph
ultrawork implement OAuth               # 自动激活 ultrawork
```

### 3.5 魔法关键词参考（第 289-307 行）

| 关键词 | 效果 |
|-------|------|
| `ultrawork`, `ulw`, `uw` | 并行代理编排 |
| `autopilot`, `build me`, `I want a` | 自动执行管道 |
| `ralph`, `don't stop`, `must complete` | 循环直到完成 |
| `ccg`, `claude-codex-gemini` | 3 模型编排 |
| `ralplan` | 共识规划 |
| `deep interview`, `ouroboros` | 苏格拉底式深度面试 |
| `code review`, `review code` | 代码审查模式 |
| `security review` | 安全专注审查 |
| `deepsearch` | 代码库搜索 |
| `deepanalyze` | 深度分析 |
| `ultrathink` | 深度推理 |
| `tdd`, `test first` | TDD 工作流 |
| `deslop`, `anti-slop` | AI 表达清理 |
| `cancelomc`, `stopomc` | 取消执行模式 |

**重要**：关键词处理有两个源头（第 308-317 行）：
- `config.jsonc` 中的 `magicKeywords` — 可定制
- `keyword-detector` hook — 硬编码（不可改）

---

## 四、Hooks 系统详解（第 320-425 行）

### 4.1 Hook 的本质（第 320-325 行）

**Hooks 是对 Claude Code 生命周期事件的反应**：

```
生命周期事件 → Hook 脚本 → OMC 逻辑 → 结果注入回 Claude Code
```

### 4.2 Claude Code 提供的 11 个生命周期事件（第 327-343 行）

| 事件 | 触发时机 | OMC 用途 |
|------|---------|---------|
| `UserPromptSubmit` | 用户提交提示 | 魔法关键词检测、技能注入 |
| `SessionStart` | 会话开始 | 初始设置、项目内存加载 |
| `PreToolUse` | 工具使用前 | 权限验证、并行执行提示 |
| `PermissionRequest` | 权限请求 | Bash 命令权限处理 |
| `PostToolUse` | 工具使用后 | 结果验证、项目内存更新 |
| `PostToolUseFailure` | 工具失败 | 错误恢复处理 |
| `SubagentStart` | Subagent 启动 | Agent 追踪 |
| `SubagentStop` | Subagent 停止 | Agent 追踪、输出验证 |
| `PreCompact` | 上下文压缩前 | 保存关键信息、项目内存持久化 |
| `Stop` | Claude 即将停止 | 持续模式强制、代码简化 |
| `SessionEnd` | 会话结束 | 会话数据清理 |

### 4.3 system-reminder 注入（第 345-362 行）

Hooks 通过 `<system-reminder>` 标签向 Claude 注入上下文：

```xml
<system-reminder>
hook success: Success
</system-reminder>
```

**注入的模式含义**：

| 模式 | 含义 |
|------|------|
| `hook success: Success` | Hook 正常运行，继续执行 |
| `hook additional context: ...` | 附加上下文信息，记住 |
| `[MAGIC KEYWORD: ...]` | 检测到魔法关键词，执行对应技能 |
| `The boulder never stops` | ralph/ultrawork 模式活跃 |

### 4.4 关键 Hook 实现（第 364-387 行）

#### keyword-detector
- **触发**：`UserPromptSubmit`
- **作用**：检测用户输入中的魔法关键词，激活对应技能

#### persistent-mode
- **触发**：`Stop`
- **作用**：当 ralph/ultrawork 模式活跃时，防止 Claude 停止直到工作验证完成

#### pre-compact
- **触发**：`PreCompact`
- **作用**：在上下文窗口压缩前，保存关键信息到 notepad

#### subagent-tracker
- **触发**：`SubagentStart`, `SubagentStop`
- **作用**：追踪当前运行的 Agent；在停止时验证输出

#### context-guard-stop
- **触发**：`Stop`
- **作用**：监控上下文使用情况，接近限制时发出警告

#### code-simplifier
- **触发**：`Stop`
- **作用**：（默认禁用）当启用时，Claude 停止时自动简化修改的文件

### 4.5 Hook 注册结构（第 389-412 行）

```json
{
  "UserPromptSubmit": [
    {
      "matcher": "*",
      "hooks": [
        {
          "type": "command",
          "command": "node scripts/keyword-detector.mjs",
          "timeout": 5
        }
      ]
    }
  ]
}
```

关键字段：
- `matcher`：Hook 响应的模式（`*` 匹配所有输入）
- `timeout`：超时时间（秒）
- `type`：类型（总是 `"command"`）

### 4.6 禁用 Hook（第 414-424 行）

```bash
# 禁用所有 hook
export DISABLE_OMC=1

# 跳过特定 hook（逗号分隔）
export OMC_SKIP_HOOKS="keyword-detector,persistent-mode"
```

---

## 五、State 管理系统详解（第 427-593 行）

### 5.1 State 系统的目的（第 427-432 行）

**State 系统保存任务进度和项目知识**，即使上下文窗口重置后也能恢复：

```
问题：上下文压缩会重置窗口
解决：将关键信息保存到 .omc/ 目录
目标：跨越上下文重置的连续性
```

### 5.2 目录结构（第 434-459 行）

```
.omc/
├── state/                         # 模式状态文件
│   ├── autopilot-state.json       # autopilot 进度
│   ├── ralph-state.json           # ralph 循环状态
│   ├── team/                      # team 任务状态
│   ├── interop/                   # 跨工具任务/消息信封
│   └── sessions/                  # 每个会话状态
│       └── {sessionId}/
├── notepad.md                     # 压缩抗性备忘录
├── project-memory.json            # 项目知识存储
├── plans/                         # 执行计划
├── notepads/                      # 每个计划的知识捕获
│   └── {plan-name}/
│       ├── learnings.md           # 发现的模式、成功方法
│       ├── decisions.md           # 架构决策和理由
│       ├── issues.md              # 问题和阻碍
│       └── problems.md            # 技术债和警告
├── prompts/                       # 持久化的提示/响应
├── autopilot/                     # autopilot 成物
│   └── spec.md
├── research/                      # 研究结果
└── logs/                          # 执行日志
```

### 5.3 Control Plane vs Data Plane（第 461-477 行）

**关键设计**：分离编排元数据和大型持久成物

```
┌────────────────────────────────────┐
│ CONTROL PLANE                      │
│ (小，快速访问)                      │
│ .omc/state/**                      │
│ ├─ 队列状态                        │
│ ├─ Worker 分配                     │
│ ├─ 会话状态                        │
│ └─ 跨工具任务/消息信封             │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│ DATA PLANE                         │
│ (大，持久成物)                      │
│ .omc/plans/, .omc/notepads/       │
│ .omc/prompts/, .omc/state/interop │
│ ├─ 计划、规范                      │
│ ├─ 提示、结果、跟踪                │
│ └─ 其他持久成物                    │
└────────────────────────────────────┘
```

**好处**：
- 调度器和状态检查保持小规模
- 复杂成物可持久存储和检查

### 5.4 Artifact 描述符和有界移交（第 478-497 行）

**原则**：用描述符替代整个有效负载

```typescript
{
  kind: "plan",              // 成物类别
  path: "/path/to/plan.md",  // 持久路径
  contentHash?: "abc123",    // 可选完整性检查
  createdAt: 1234567890,     // 创建时间戳
  producer: "planner",       // 所有者/生产者
  sizeBytes?: 5000,          // 可选有效负载大小
  retention: "project",      // 生命周期提示
  expiresAt?: 1234567890     // 可选过期时间
}
```

**有界移交规则**：
1. 小有效负载内联（如果允许）
2. 大有效负载用描述符 + 摘要
3. 保存所有者/保留元数据

### 5.5 Notepad（第 499-521 行）

**文件**：`.omc/notepad.md`

**目的**：在上下文压缩后生存的备忘录

**工作机制**：
1. 在 `PreCompact` 事件，重要信息保存到 notepad
2. 压缩后，notepad 内容重新注入到上下文
3. Agent 使用 notepad 恢复之前的上下文

**MCP 工具**：

| 工具 | 描述 |
|------|------|
| `notepad_read` | 读取 notepad 内容 |
| `notepad_write_priority` | 写高优先级备忘录（永久保留） |
| `notepad_write_working` | 写工作备忘录 |
| `notepad_write_manual` | 写手动备忘录 |
| `notepad_prune` | 清理旧备忘录 |
| `notepad_stats` | 查看 notepad 统计 |

### 5.6 Project Memory（第 523-542 行）

**文件**：`.omc/project-memory.json`

**目的**：跨会话持久存储项目级知识

**生命周期集成**：
- `SessionStart`：加载项目内存，注入到上下文
- `PostToolUse`：从工具结果提取项目知识并保存
- `PreCompact`：在上下文压缩前保存项目内存

**MCP 工具**：

| 工具 | 描述 |
|------|------|
| `project_memory_read` | 读取项目内存 |
| `project_memory_write` | 覆盖整个项目内存 |
| `project_memory_add_note` | 添加笔记 |
| `project_memory_add_directive` | 添加指令 |

### 5.7 Per-Plan Knowledge Capture（第 549-562 行）

**路径**：`.omc/notepads/{plan-name}/`

**文件**：

| 文件 | 内容 |
|------|------|
| `learnings.md` | 发现的模式、成功方法 |
| `decisions.md` | 架构决策和理由 |
| `issues.md` | 问题和阻碍 |
| `problems.md` | 技术债和警告 |

**重要**：所有条目自动带时间戳

### 5.8 持久内存标签（第 578-587 行）

```xml
<!-- 保留 7 天 -->
<remember>API endpoint changed to /v2</remember>

<!-- 永久保留 -->
<remember priority>Never access production DB directly</remember>
```

| 标签 | 保留期 |
|------|--------|
| `<remember>` | 7 天 |
| `<remember priority>` | 永久 |

---

## 六、验证协议（第 595-610 行）

验证模块确保工作完成且有证据：

**标准检查**：

| 检查 | 描述 |
|------|------|
| BUILD | 编译通过 |
| TEST | 所有测试通过 |
| LINT | 无 lint 错误 |
| FUNCTIONALITY | 功能按预期工作 |
| ARCHITECT | Opus 级别审查批准 |
| TODO | 所有任务完成 |
| ERROR_FREE | 无未解决错误 |

**要求**：
- 证据必须是新鲜的（5 分钟内）
- 必须包含实际的命令输出

---

## 七、关键设计原则总结

### 7.1 分离关注点

```
Agents → 做什么
Skills → 怎么做
Hooks → 何时做
State → 进度追踪
```

### 7.2 职责清晰

```
explore ≠ analyst ≠ planner ≠ architect ≠ executor ≠ verifier
  │       │       │       │       │       │
  └─ 每个 Agent 只有一个职责，不重叠
```

### 7.3 成本优化

```
Haiku (便宜) → 快速查询
Sonnet (中等) → 代码实现
Opus (昂贵) → 深度分析

按复杂度选择模型，避免过度使用昂贵模型
```

### 7.4 跨上下文重置的连续性

```
notepad → 短期记忆（7 天）
project-memory → 长期记忆（跨会话）
state/ → 操作状态
```

### 7.5 可组合性

```
Skills 可组合：
[Execution] + [Enhancement 1] + [Enhancement 2] + [Optional Guarantee]

例如：
default + ultrawork + git-master + ralph
```

### 7.6 自动化的精准度

```
从用户输入
  → 自动检测任务类型
  → 自动选择 Agent 组合
  → 自动选择模型
  → 自动激活 Skills
  → 执行
```

---

## 八、与其他文档的关联

该文档是 OMC 文档体系的核心：

```
ARCHITECTURE.md (本文)
  ├─ Agent 系统详解
  ├─ Skills 系统详解
  ├─ Hooks 系统详解
  └─ State 管理详解
       ↓ (详细参考)
REFERENCE.md
  └─ 完整 API 参考
  
FEATURES.md
  └─ 内部 API 文档
  
README.md
  └─ 用户指南
```

---

## 九、ARCHITECTURE.md 的价值

| 层面 | 价值 |
|------|------|
| **架构师** | 理解整体系统设计和职责分离 |
| **开发者** | 知道如何扩展（新 Agent、新 Skill、新 Hook） |
| **贡献者** | 理解设计原则，确保新代码符合架构 |
| **用户** | 理解系统如何工作，更好地利用 OMC |

---

## 十、核心洞察

### 10.1 四大系统的互动

```
Hooks (事件)
  ↓
 Skills (行为)
  ↓
Agents (执行)
  ↓
State (记忆)
  ↓
  ... 循环 ...
```

### 10.2 OMC 的核心竞争力

1. **职责明确**：每个 Agent 做一件事，做得很好
2. **自动化**：从需求自动推导出执行计划
3. **成本优化**：按复杂度选择模型
4. **可恢复**：跨上下文重置的连续性
5. **可组合**：Skills 和 Agents 灵活组合

### 10.3 与其他编排系统的区别

```
传统编排：
Orchestrator → Agent A → Agent B → Agent C → Done

OMC 编排：
用户输入 → Hook 检测 → Skill 选择 → Agent 组合 → 
上下文重置? → 恢复 State → 继续工作 → Done
```

---

## 总结

**ARCHITECTURE.md 是 OMC 的蓝图**，阐述了：

1. **四大核心系统**（Hooks, Skills, Agents, State）如何协同工作
2. **19 个 Agent** 的职责和模型选择
3. **6 个核心 Skill**（autopilot, ralph, ultrawork, team, ccg, ralplan）
4. **11 个生命周期事件**和 Hook 机制
5. **State 管理**如何实现跨上下文重置的连续性
6. **成本优化**和**验证协议**

**对理解 OMC 来说必读**。

