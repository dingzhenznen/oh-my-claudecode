# OMC 架构可视化与快速参考指南

## 一、系统架构全景图

### 1.1 完整执行管道

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OMC 完整执行管道                                   │
└─────────────────────────────────────────────────────────────────────┘

User Input
   │
   ├─ "ultrawork refactor API"
   │
   ▼
┌─────────────────────────────────────────┐
│ EVENT DETECTION (Hooks)                 │
│ - UserPromptSubmit 事件触发              │
│ - keyword-detector 检测关键词            │
│ - [MAGIC KEYWORD: ultrawork] 注入       │
└─────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────┐
│ SKILL INJECTION (Skills)                │
│ - ultrawork 技能激活                    │
│ - default 执行层启动                    │
│ - git-master 增强层启动                 │
│ - 组合：default + ultrawork + git-master│
└─────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────┐
│ AGENT EXECUTION (Agents)                │
│ ┌─────────────┐  ┌──────────────┐      │
│ │ explore     │  │ executor     │      │
│ │ (Haiku)     │  │ (Sonnet)     │      │
│ └─────────────┘  └──────────────┘      │
│        ▼                ▼               │
│ 文件地图 ────────→ 代码修改             │
│                                        │
│ ┌──────────────────────────────┐      │
│ │ git-master (Sonnet)          │      │
│ │ 原子提交                      │      │
│ └──────────────────────────────┘      │
│        ▼                               │
│ 创建提交，推送代码                     │
└─────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────┐
│ STATE MANAGEMENT (State)                │
│ - .omc/state/ultrawork-state.json       │
│ - 保存进度，以便上下文重置后恢复        │
│ - 若上下文压缩，从 state 恢复           │
└─────────────────────────────────────────┘
   │
   ▼
Output / Done
```

---

## 二、19 个 Agent 的完整地图

### 2.1 Agent 树形结构

```
OMC Agents (19 total)
│
├─ BUILD/ANALYSIS LANE (8个)
│  ├─ explore (Haiku)           ← 入口点：发现文件
│  ├─ analyst (Opus)             ← 分析需求
│  ├─ planner (Opus)             ← 创建计划
│  ├─ architect (Opus)           ← 设计审查
│  ├─ debugger (Sonnet)          ← 根本原因分析
│  ├─ executor (Sonnet)          ← 代码实现
│  ├─ verifier (Sonnet)          ← 完成验证
│  └─ tracer (Sonnet)            ← 证据追踪
│
├─ REVIEW LANE (2个)
│  ├─ security-reviewer (Sonnet) ← 安全审查
│  └─ code-reviewer (Opus)       ← 代码审查
│
├─ DOMAIN LANE (9个)
│  ├─ test-engineer (Sonnet)     ← 测试策略
│  ├─ designer (Sonnet)          ← UI/UX 设计
│  ├─ writer (Haiku)             ← 文档
│  ├─ qa-tester (Sonnet)         ← 集成测试
│  ├─ scientist (Sonnet)         ← 数据分析
│  ├─ git-master (Sonnet)        ← Git 操作
│  ├─ document-specialist (Sonnet) ← 外部研究
│  ├─ code-simplifier (Opus)     ← 代码简化
│  └─ [未使用的域 Agent]
│
└─ COORDINATION LANE (1个)
   └─ critic (Opus)              ← 计划审查
```

### 2.2 Build/Analysis Lane 的工作流

```
                    Build/Analysis Lane

                         ┌─────────┐
                         │ explore │ (Haiku, cheap)
                         │ 发现    │
                         └────┬────┘
                              │ 输出：文件地图
                              ▼
                         ┌─────────┐
                         │ analyst │ (Opus)
                         │ 分析    │
                         └────┬────┘
                              │ 输出：需求缺口
                              ▼
                         ┌─────────┐
                         │ planner │ (Opus)
                         │ 规划    │
                         └────┬────┘
                              │ 输出：任务列表
                              ▼
                         ┌─────────┐
                         │ critic  │ (Opus)
                         │ 审查    │
                         └────┬────┘
                              │ 输出：批准/拒绝
                              ▼
                         ┌─────────┐
                         │executor │ (Sonnet)
                         │ 实施    │
                         └────┬────┘
                              │ 输出：代码修改
                              ▼
                         ┌─────────┐
                         │verifier │ (Sonnet)
                         │ 验证    │
                         └─────────┘
```

### 2.3 模型分布热力图

```
成本高              Opus (昂贵)
  │   ┌──────────────────────────────────┐
  │   │ architect, planner, critic       │
  │   │ code-reviewer, code-simplifier  │
  │   └──────────────────────────────────┘
  │
  │   ┌──────────────────────────────────┐
  │   │ Sonnet (中等成本)                │
  │   │ debugger, executor, verifier    │
  │   │ test-engineer, designer, qa-    │
  │   │ tester, scientist, git-master   │
  │   │ security-reviewer, tracer,      │
  │   │ document-specialist             │
  │   └──────────────────────────────────┘
  │
  │   ┌──────────────────────────────────┐
成本低 │   │ Haiku (便宜)                   │
  │   │ explore, writer                │
  │   └──────────────────────────────────┘
  │
  └─────────────────────────────────────
    质量 ──────────────────────────────→
```

---

## 三、Skills 系统的完整图解

### 3.1 核心 6 个 Skills

```
┌──────────────────────────────────────────────────┐
│ 1. autopilot                                     │
│ ─────────────────────────────────────────────   │
│ 触发：autopilot, build me, I want a, e2e        │
│ 流程：需求 → 计划 → 实施 → 测试 → 提交          │
│ 保证：5 阶段完整管道                            │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 2. ralph                                         │
│ ─────────────────────────────────────────────   │
│ 触发：ralph, don't stop, must complete          │
│ 流程：PRD → Progress → Verify → (失败?) 循环    │
│ 保证：验证完成才停止                             │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 3. ultrawork                                     │
│ ─────────────────────────────────────────────   │
│ 触发：ultrawork, ulw                            │
│ 流程：最大并行启动 N 个 Agent                   │
│ 保证：高吞吐量                                   │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 4. team                                          │
│ ─────────────────────────────────────────────   │
│ 用法：/oh-my-claudecode:team 3:executor "..."   │
│ 流程：plan → prd → exec → verify → fix (循环)   │
│ 保证：多 Agent 协作                             │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 5. ccg                                           │
│ ─────────────────────────────────────────────   │
│ 触发：ccg, claude-codex-gemini                  │
│ 流程：扇出 Codex + Gemini → Claude 综合         │
│ 保证：3 模型协作                                │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 6. ralplan                                       │
│ ─────────────────────────────────────────────   │
│ 触发：ralplan                                   │
│ 流程：Planner + Architect + Critic 循环         │
│ 保证：达成共识                                   │
└──────────────────────────────────────────────────┘
```

### 3.2 Skills 组合矩阵

```
任务类型              推荐技能组合
──────────────────────────────────────────────

功能构建             default
                    ↓
多文件重构           default + ultrawork

重构 + 提交          default + git-master

并行构建             default + ultrawork

循环直到完成         default + ralph

3 模型审查           default + ccg

完整项目             autopilot

多 Agent 协作        team
```

---

## 四、Hooks 系统的事件流

### 4.1 生命周期事件 → Hook 触发

```
Claude Code 生命周期                Hook 反应
─────────────────────────────────────────────

User Submits Prompt
│
├─ UserPromptSubmit ──────→ keyword-detector
│                          └─ 检测 magic keywords
│                          └─ 注入 [MAGIC KEYWORD: X]
│
Session Starts
├─ SessionStart ──────────→ project-memory-load
│                          └─ 加载项目内存
│
Tool About to Use
├─ PreToolUse ────────────→ permission-check
│                          └─ 验证权限
│
Tool Used
├─ PostToolUse ───────────→ project-memory-save
│                          └─ 提取和保存知识
│
Tool Failed
├─ PostToolUseFailure ────→ error-recovery
│                          └─ 恢复处理
│
Subagent Starts/Stops
├─ SubagentStart ─────────→ subagent-tracker
├─ SubagentStop ──────────→ validate-output
│
Context About to Compact
├─ PreCompact ────────────→ pre-compact
│                          └─ 保存关键信息到 notepad
│
Claude About to Stop
├─ Stop ──────────────────→ persistent-mode
│                          │ └─ ralph/ultrawork 模式检查
│                          ├─ code-simplifier
│                          │ └─ 代码简化（可选）
│                          └─ context-guard-stop
│                             └─ 警告上下文限制
│
Session Ends
└─ SessionEnd ────────────→ cleanup
                           └─ 会话数据清理
```

### 4.2 Hook → Claude 的信息注入

```
OMC Hooks
    ├─ 检测事件
    ├─ 执行逻辑
    └─ 生成 system-reminder
           │
           ├─ hook success: Success
           │  └─ Claude 继续执行
           │
           ├─ [MAGIC KEYWORD: ultrawork]
           │  └─ Claude 激活 ultrawork 技能
           │
           ├─ hook additional context: ...
           │  └─ Claude 记住额外信息
           │
           └─ The boulder never stops
              └─ ralph/ultrawork 模式，不停止
```

---

## 五、State 管理的数据流

### 5.1 State 保存的信息流

```
                     State Management Flow

                         执行中
                          │
                    ┌─────▼─────┐
                    │  Boulder  │
                    │ State     │ ← 追踪进度
                    └─────┬─────┘
                          │
                ┌─────────┴─────────┐
                │                   │
       ┌────────▼────────┐  ┌───────▼───────┐
       │ notepad.md      │  │ project-      │
       │ (7 天)          │  │ memory.json   │
       │                 │  │ (永久)        │
       │ • 当前任务      │  │               │
       │ • 最后进度      │  │ • API 设计    │
       │ • 已知陷阱      │  │ • 项目规则    │
       │ • 上一步结果    │  │ • 决策理由    │
       └────────┬────────┘  └───────┬───────┘
                │                   │
                └─────────┬─────────┘
                          │
                    ┌─────▼─────┐
              PreCompact 事件
                    │
              上下文压缩
                    │
              ┌─────▼─────┐
              │ Notepad   │
              │ 内容重新  │
              │ 注入      │
              └───────────┘
                    │
                    ▼
              Claude 恢复上下文
              继续工作
```

### 5.2 .omc/ 目录的逻辑

```
.omc/
├─ Control Plane (快速访问)
│  ├─ state/
│  │  ├─ autopilot-state.json (操作状态)
│  │  ├─ ralph-state.json
│  │  ├─ team/ (团队状态)
│  │  ├─ interop/ (跨工具信息)
│  │  └─ sessions/ (会话隔离)
│  │
│  └─ notepad.md (压缩后恢复)
│
├─ Data Plane (持久成物)
│  ├─ plans/ (执行计划)
│  ├─ notepads/ (知识捕获)
│  │  └─ {plan-name}/
│  │     ├─ learnings.md
│  │     ├─ decisions.md
│  │     ├─ issues.md
│  │     └─ problems.md
│  │
│  ├─ prompts/ (持久化提示)
│  ├─ autopilot/
│  │  └─ spec.md
│  ├─ research/ (研究结果)
│  └─ logs/ (执行日志)
│
└─ project-memory.json (全局知识)
```

---

## 六、快速决策树

### 6.1 选择 Agent 的决策树

```
What do you need?
│
├─ "Find files in the codebase"
│  └─ → explore (Haiku)
│
├─ "Analyze requirements"
│  └─ → analyst (Opus)
│
├─ "Create execution plan"
│  └─ → planner (Opus)
│
├─ "Review plan quality"
│  └─ → critic (Opus)
│
├─ "Implement code changes"
│  ├─ Simple → executor (Sonnet)
│  └─ Complex → executor (Opus)
│
├─ "Fix a bug"
│  ├─ Simple → debugger (Sonnet)
│  └─ Complex → architect (Opus)
│
├─ "Verify completion"
│  └─ → verifier (Sonnet)
│
├─ "Review code quality"
│  └─ → code-reviewer (Opus)
│
├─ "Security audit"
│  └─ → security-reviewer (Sonnet)
│
├─ "Write tests"
│  └─ → test-engineer (Sonnet)
│
├─ "UI/UX design"
│  └─ → designer (Sonnet)
│
├─ "Documentation"
│  └─ → writer (Haiku)
│
├─ "Trace root cause"
│  └─ → tracer (Sonnet)
│
├─ "Git operations"
│  └─ → git-master (Sonnet)
│
├─ "External research"
│  └─ → document-specialist (Sonnet)
│
├─ "Simplify code"
│  └─ → code-simplifier (Opus)
│
└─ "Data analysis"
   └─ → scientist (Sonnet)
```

### 6.2 选择 Skill 的决策树

```
What is your task?
│
├─ "Autonomous end-to-end"
│  └─ → /autopilot
│
├─ "Parallel execution"
│  └─ → /ultrawork
│
├─ "Loop until verified"
│  └─ → /ralph
│
├─ "Multiple agents coordinate"
│  └─ → /team N:agent-type
│
├─ "3-model comparison"
│  └─ → /ccg
│
├─ "Consensus planning"
│  └─ → /ralplan
│
└─ (None, just use default)
   └─ → default skill
```

---

## 七、成本对比表

### 7.1 按 Token 成本排序

```
排序     模型      成本比  典型用途
─────────────────────────────────────────
1      Haiku     1.0x   快速查询、文档
2      Sonnet    3.0x   代码实现、调试
3      Opus      10.0x  架构、深度分析

成本计算示例（假设 1K Haiku = $0.01）：
─────────────────────────────────────────
explore(Haiku, 2K)     = $0.02
executor(Sonnet, 5K)   = $0.15
planner(Opus, 5K)      = $0.50

总成本：$0.67
（如果全用 Opus，10K = $1.00，节省 33%）
```

### 7.2 按任务复杂度选择模型

```
复杂度          推荐模型         理由
──────────────────────────────────────
简单查询        Haiku           快速便宜
简单实现        Sonnet          质量够，成本低
复杂实现        Opus            需要深度思考
策略决策        Opus            需要广泛考虑
```

---

## 八、关键概念速查表

| 概念 | 定义 | 例子 |
|------|------|------|
| **Agent** | 专门化的 Claude 实例 | explore, executor, verifier |
| **Skill** | 行为注入 | autopilot, ralph, ultrawork |
| **Hook** | 生命周期事件反应 | keyword-detector, persistent-mode |
| **State** | 进度和知识持久化 | .omc/state/, notepad.md |
| **Magic Keyword** | 触发 Skill 的关键词 | "autopilot", "ralph", "ultrawork" |
| **Lane** | Agent 分组 | Build/Analysis, Review, Domain |
| **Model Routing** | 按复杂度选模型 | Haiku < Sonnet < Opus |
| **Boulder State** | 工作块进度 | 当前计划、任务列表、完成状态 |
| **Notepad** | 压缩后恢复的备忘 | 重要信息在上下文重置后存活 |
| **Project Memory** | 跨会话知识 | 永久记住项目规则、决策 |

---

## 九、快速参考命令

```bash
# 调用 Skill
/oh-my-claudecode:autopilot build me a todo app
/oh-my-claudecode:ralph refactor the auth module
/oh-my-claudecode:ultrawork implement OAuth
/oh-my-claudecode:team 3:executor "fullstack app"
/oh-my-claudecode:ccg review this code
/oh-my-claudecode:ralplan this feature

# Magic 关键词（自动激活）
autopilot build me a REST API
ralph: refactor authentication
ultrawork implement user management

# 管理 Hook
export DISABLE_OMC=1  # 禁用所有 Hook
export OMC_SKIP_HOOKS="keyword-detector,persistent-mode"  # 跳过特定 Hook

# 取消执行
/oh-my-claudecode:cancel

# 配置
/oh-my-claudecode:omc-setup
/oh-my-claudecode:omc-doctor
```

---

## 总结

OMC 架构的核心：

```
Simple Input → Automatic Inference → Complex Execution → Continuous Recovery

"ultrawork refactor API"
    ↓
[Haiku, Sonnet, Opus 智能组合]
    ↓
[explore, executor, git-master 并行执行]
    ↓
[状态保存，即使上下文重置也能恢复]
```

