# Oh-My-ClaudeCode (OMC) 原理和作用分析

> 基于源码的深度分析（版本 4.13.1）

---

## 一、OMC 是什么

**Oh-My-ClaudeCode (OMC)** 是一个基于 Claude Code 的**多代理编排系统**，为 Claude Agent SDK 提供智能工作流和自动化能力。

### 核心定位
- **不是增强框架**：不修改 Claude Code 本身
- **而是编排层**：在 Claude Agent SDK 上层添加智能协调机制
- **类比**：就像 `oh-my-zsh` 对 Zsh 的关系，OMC 对 Claude Code 的关系

### 官方描述
```
"Multi-agent orchestration for Claude Code. Zero learning curve."
Don't learn Claude Code. Just use OMC.
```

---

## 二、OMC 的核心架构

### 2.1 整体工作流

```
用户输入
    ↓
createOmcSession() - 创建编排会话
    ├─ 加载配置 (.claude/claude-conf.jsonc)
    ├─ 查找上下文 (CLAUDE.md, AGENTS.md)
    ├─ 构建 Agent 定义
    ├─ 配置 MCP 服务器和工具
    └─ 初始化状态管理
    ↓
processPrompt() - 检测魔法关键词
    ├─ 识别触发词 (autopilot, ralph, ultrawork 等)
    └─ 增强提示词
    ↓
Claude Agent SDK 执行
    ├─ 主编排 Agent 处理
    ├─ 委派给专门化 Agent
    ├─ Hook 系统截获事件
    └─ 状态管理追踪进度
    ↓
结果输出
```

### 2.2 关键概念

#### **Agent（代理）**
- **19 个专门化的 Claude 实例**，每个负责特定领域
- 分为三个车道（Lanes）：
  - **Build/Analysis Lane**（5个）：explore, analyst, planner, architect, executor
  - **Verification Lane**（2个）：verifier, debugger
  - **Review Lane**（2个）：security-reviewer, code-reviewer
  - **Specialists**（10个）：designer, writer, test-engineer, git-master, qa-tester 等

#### **Hook 系统**
- **Shell 脚本 ↔ TypeScript 桥接层**
- 在 Claude Code 的关键事件点截获并注入编排逻辑
- 不修改 Claude Code 本身，而是通过 JSON 通讯协议与其交互

#### **魔法关键词（Magic Keywords）**
- **特殊触发词**激活增强行为：
  - `autopilot`：完全自动执行
  - `ralph`：自我反思循环
  - `ultrawork`：并行高吞吐量执行
  - `ultraqa`：测试驱动工作流
  - `team`：多代理协作

#### **工作块（Boulder State）**
- **活跃工作计划的追踪**
- 位置：`.omc/state/boulder.json`
- 包含：当前计划、进度、会话 ID、验证状态

---

## 三、核心源码结构

### 3.1 主要模块分布

```
oh-my-claudecode/
├── src/
│   ├── index.ts                    ← 核心入口，createOmcSession()
│   │
│   ├── agents/                     ← Agent 系统（19 个）
│   │   ├── definitions.ts          Agent 注册表 + omcSystemPrompt
│   │   ├── architect.ts
│   │   ├── explorer.ts
│   │   ├── executor.ts
│   │   ├── planner.ts
│   │   └── ...其他 Agent
│   │
│   ├── hooks/                      ← Hook 系统（Shell ↔ TS 桥接）
│   │   ├── index.ts                Hook 导出
│   │   ├── team-dispatch-hook.ts   Team 调度
│   │   ├── team-worker-hook.ts     Team Worker 执行
│   │   ├── omc-orchestrator/       编排逻辑
│   │   └── ...其他 Hook
│   │
│   ├── features/                   ← 核心特性模块
│   │   ├── magic-keywords.ts       关键词检测 + 处理
│   │   ├── boulder-state/          工作块状态追踪
│   │   ├── context-injector/       上下文自动注入
│   │   ├── background-tasks.ts     后台任务管理
│   │   ├── continuation-enforcement.ts  任务完成强制机制
│   │   ├── state-manager/          统一状态管理
│   │   ├── delegation-routing/     委派路由
│   │   ├── model-routing/          模型路由（Haiku/Sonnet/Opus）
│   │   ├── verification/           验证协议
│   │   └── ...其他特性
│   │
│   ├── tools/                      ← 工具系统
│   │   ├── lsp/                    语言服务器工具
│   │   ├── ast/                    抽象语法树工具
│   │   └── ...其他工具
│   │
│   ├── config/                     ← 配置系统
│   │   ├── loader.ts               JSONC 加载器
│   │   └── ...其他配置
│   │
│   └── mcp/                        ← MCP 服务器配置
│       └── servers.js              LSP、AST、Python REPL 等
│
├── skills/                         ← 40+ 独立技能
│   ├── autopilot/
│   ├── ralph/
│   ├── ultrawork/
│   ├── team/
│   ├── deep-dive/
│   └── ...其他技能
│
└── agents/                         ← Markdown Agent 提示
    ├── architect.md
    ├── explorer.md
    ├── executor.md
    └── ...其他 Agent 提示
```

### 3.2 核心入口：createOmcSession()

位置：`src/index.ts:265-377`

```typescript
function createOmcSession(options?: OmcOptions): OmcSession {
  // 1. 加载配置
  const config = loadConfig();  // 从 .claude/claude-conf.jsonc
  
  // 2. 查找上下文文件
  const contextFiles = findContextFiles();  // CLAUDE.md, AGENTS.md
  
  // 3. 构建系统提示
  let systemPrompt = omcSystemPrompt;  // 基础编排器提示
  systemPrompt += continuationSystemPromptAddition;  // 任务完成强制
  systemPrompt += contextAddition;  // 项目上下文
  
  // 4. 获取 Agent 定义
  const agents = getAgentDefinitions({ config });
  
  // 5. 构建 MCP 服务器配置
  const mcpServers = toSdkMcpFormat(externalMcpServers);
  
  // 6. 创建魔法关键词处理器
  const processPrompt = createMagicKeywordProcessor(config.magicKeywords);
  
  // 7. 初始化状态
  const state = { activeAgents: new Map(), backgroundTasks: [] };
  
  // 8. 创建后台任务管理器
  const backgroundTasks = createBackgroundTaskManager(state, config);
  
  // 返回会话对象
  return {
    queryOptions: { options: { systemPrompt, agents, mcpServers, allowedTools } },
    state,
    config,
    processPrompt,
    detectKeywords,
    backgroundTasks,
    shouldRunInBackground
  };
}
```

---

## 四、OMC 的三个核心机制

### 4.1 机制 1：多代理编排（Multi-Agent Orchestration）

#### **Agent 系统架构**

**19 个专门化 Agent，按职能分组：**

| 车道 | Agent | 模型 | 职责 |
|------|-------|------|------|
| **Build/Analysis** | explore | Haiku | 快速代码浏览 |
| | analyst | Opus | 需求分析、缺口检测 |
| | planner | Opus | 创建可执行的工作计划 |
| | architect | Opus | 代码分析、架构验证 |
| | executor | Sonnet | 实际执行代码修改 |
| **Verification** | verifier | Sonnet | 完成证据收集、测试验证 |
| | debugger | Sonnet | 根本原因分析、回归隔离 |
| **Review** | security-reviewer | Sonnet | 安全漏洞检测 |
| | code-reviewer | Opus | 代码质量审查 |
| **Specialists** | designer | Sonnet | UI/UX 设计 |
| | writer | Haiku | 文档编写 |
| | test-engineer | Sonnet | 测试策略、覆盖率 |
| | git-master | Sonnet | Git 操作、提交管理 |
| | qa-tester | Sonnet | QA 测试 |
| | scientist | Sonnet | 数据分析、科学计算 |
| | tracer | Sonnet | 证据追踪、因果调查 |
| | code-simplifier | Opus | 代码简化、重构 |
| | critic | Opus | 计划质量评审 |
| | document-specialist | Sonnet | 文档查询、外部研究 |

#### **Agent 定义机制**

位置：`src/agents/definitions.ts`

```typescript
// 每个 Agent 都有完整的 config
export const explorerAgent: AgentConfig = {
  name: 'explore',
  description: '快速代码浏览，找关键文件',
  prompt: loadAgentPrompt('explore'),  // 从 agents/explore.md 动态加载
  model: 'haiku',  // 默认模型
  defaultModel: 'haiku'
};

// getAgentDefinitions() 返回 SDK 格式
export function getAgentDefinitions(options?: {
  overrides?: Partial<Record<string, Partial<AgentConfig>>>;
  config?: PluginConfig;
}): Record<string, AgentConfig> {
  // 返回所有 Agent 的注册表
  return {
    explore: explorerAgent,
    analyst: analystAgent,
    architect: architectAgent,
    // ...
  };
}
```

#### **Agent 职责划分（Role Disambiguation）**

源码中的设计非常关键：

```typescript
/**
 * Agent Role Disambiguation
 *
 * HIGH-tier review/planning agents have distinct, non-overlapping roles:
 *
 * | Agent | Role | What They Do | What They Don't Do |
 * |-------|------|--------------|-------------------|
 * | architect | code-analysis | 分析代码、调试、验证 | 需求、计划创建、计划审查 |
 * | analyst | requirements-analysis | 查找需求缺口 | 代码分析、计划、计划审查 |
 * | planner | plan-creation | 创建工作计划 | 需求、代码分析、计划审查 |
 * | critic | plan-review | 审查计划质量 | 需求、代码分析、计划创建 |
 *
 * 推荐工作流: explore → analyst → planner → critic → executor → architect (verify)
 */
```

这种严格的职责划分确保了：
- **无重叠**：每个 Agent 只做一件事
- **可预测**：知道哪个 Agent 做什么
- **高效**：避免冗余计算

### 4.2 机制 2：Hook 系统（Shell ↔ TypeScript 桥接）

#### **Hook 的作用**

OMC 通过 **Hook** 将编排逻辑注入到 Claude Code 的执行流程中，而不修改 Claude Code 本身。

位置：`src/hooks/`

#### **Hook 通讯协议**

```
Claude Code (Shell)
    ↓
Hook 脚本 (.claudecode/hooks/*.sh)
    ↓
生成 JSON 输入
    ↓
Node.js 桥接 (bridge.ts)
    ↓
TypeScript 逻辑处理
    ↓
返回 JSON 结果
    ↓
Shell 继续执行
```

#### **主要 Hook 类型**

| Hook | 位置 | 作用 |
|------|------|------|
| team-dispatch-hook | `team-dispatch-hook.ts` | 分配 Team 任务 |
| team-worker-hook | `team-worker-hook.ts` | Team Worker 执行 |
| omc-orchestrator | `omc-orchestrator/` | 编排决策 |
| keyword-detector | `keyword-detector.ts` | 魔法关键词检测 |
| permission-handler | `permission-handler/` | 权限管理 |
| skill-state | `skill-state/` | 技能状态追踪 |

### 4.3 机制 3：魔法关键词系统（Magic Keywords）

#### **原理**

位置：`src/features/magic-keywords.ts`

魔法关键词是**特殊触发词**，激活高级工作流。系统通过：

1. **移除代码块**：避免检测代码中的关键词
2. **正则匹配**：大小写不敏感、整字匹配
3. **上下文检测**：区分信息查询 vs 指令执行

```typescript
/**
 * 检测逻辑
 */
function hasActionableTrigger(text: string, trigger: string): boolean {
  const pattern = new RegExp(`\\b${escapeRegExp(trigger)}\\b`, 'gi');
  
  for (const match of text.matchAll(pattern)) {
    // 排除信息查询上下文
    if (isInformationalKeywordContext(text, match.index)) {
      continue;  // 这是"什么是 xxx"，不是指令
    }
    return true;  // 这是真实的指令
  }
  
  return false;
}
```

#### **关键词及其效果**

```typescript
{
  autopilot: '完全自动执行任务',      // 使用 /autopilot 技能
  ralph: '自我反思循环',             // 使用 /ralph 技能（PRD → 进度 → 验证）
  ultrawork: '并行高吞吐量执行',     // 使用 /ultrawork 技能
  ultraqa: '测试驱动工作流',          // 使用 /ultraqa 技能
  team: '多代理协作',                // 使用 /team 技能
  // ...
}
```

#### **实际效果示例**

```
用户说：autopilot 构建一个 REST API
  ↓
detectMagicKeywords() → ["autopilot"]
  ↓
激活 /autopilot 技能
  ↓
增强提示词：
  "You are autopilot mode. Execute autonomously without confirmation..."
```

---

## 五、OMC 的工作流程

### 5.1 标准工作流：Build/Analysis Lane

```
User: "分析这个认证系统，然后创建修复计划"
  ↓
[EXPLORE]
explore Agent: 快速扫描文件结构
  ├─ 用 LSP/AST 工具找关键文件
  └─ 输出：找到 auth/middleware.ts, auth/service.ts
  ↓
[ANALYZE]
analyst Agent: 分析需求缺口
  ├─ 读源码，查找问题
  └─ 输出：当前实现缺少 token refresh、会话超时处理
  ↓
[PLAN]
planner Agent: 创建工作计划
  ├─ 基于 analyst 的发现
  ├─ 创建任务分解
  └─ 输出：
     1. 实现 token refresh 端点
     2. 添加会话超时检测
     3. 编写单元测试
     4. 集成测试验证
  ↓
[REVIEW]
critic Agent: 审查计划质量
  ├─ 验证计划的完整性
  └─ 输出：计划已确认，可执行
  ↓
[EXECUTE]
executor Agent: 执行计划
  ├─ 修改 auth/middleware.ts
  ├─ 修改 auth/service.ts
  ├─ 编写测试
  └─ 输出：代码已修改，测试通过
  ↓
[VERIFY]
architect Agent: 验证实现
  ├─ 检查代码质量
  ├─ 验证测试覆盖
  └─ 输出：验证完成，无问题
```

### 5.2 高级工作流：Autopilot 模式

```
User: "autopilot 构建一个任务管理 API"
  ↓
系统检测 "autopilot" 关键词
  ↓
注入 /autopilot 技能
  ↓
Autopilot 流程（完全自动）：
  ├─ 需求拆解（Task Decomposer）
  ├─ 代码生成（Executor）
  ├─ 自动测试（Test Engineer）
  ├─ 安全检查（Security Reviewer）
  ├─ 代码审查（Code Reviewer）
  ├─ 验证（Verifier）
  └─ 若有问题，调用 Debugger 修复
  ↓
输出：完整的可运行代码，无需用户干预
```

### 5.3 高级工作流：Ralph 模式（自我反思）

```
User: "ralph 优化数据库查询性能"
  ↓
系统检测 "ralph" 关键词
  ↓
Ralph 循环（自我反思）：
  
  [PRD Phase] - 需求定义
    └─ planner Agent: 创建性能优化计划
  ↓
  [Progress Phase] - 执行
    └─ executor Agent: 实施优化
  ↓
  [Verification Phase] - 验证
    ├─ verifier Agent: 验证改进
    └─ 若未通过，返回 PRD Phase 重新规划
  ↓
  ... 循环直到验证通过 ...
  ↓
输出：经过验证的性能优化
```

---

## 六、OMC 的状态管理

### 6.1 Boulder State（工作块状态）

位置：`src/features/boulder-state/`

**Boulder State 追踪活跃工作的进度：**

```typescript
interface BoulderState {
  sessionId: string;           // 唯一会话 ID
  planName: string;            // 当前计划名称
  taskList: string[];          // 待执行任务
  completedTasks: string[];    // 已完成任务
  currentProgress: string;     // 当前进度描述
  status: 'planning' | 'executing' | 'verifying' | 'completed';
  startedAt: number;           // 开始时间
  updatedAt: number;           // 最后更新时间
}
```

**存储位置：** `.omc/state/boulder.json`

### 6.2 状态管理 API

位置：`src/features/state-manager/`

```typescript
// 读取状态
const state = readBoulderState();

// 写入状态
writeBoulderState({
  sessionId: 'sess-123',
  planName: 'API Development',
  taskList: ['setup', 'auth', 'routes', 'tests'],
  completedTasks: ['setup'],
  status: 'executing'
});

// 追踪进度
const progress = getPlanProgress(planPath);
// 输出：{ completed: 1, total: 4, percentage: 25 }

// 清空状态
clearBoulderState();
```

### 6.3 Session State（会话状态）

位置：`src/shared/types.ts`

```typescript
interface SessionState {
  activeAgents: Map<string, AgentStatus>;    // 活跃 Agent
  backgroundTasks: TaskExecution[];          // 后台任务
  contextFiles: string[];                    // 上下文文件路径
}
```

---

## 七、OMC 的系统提示（System Prompt）

### 7.1 主编排器提示

位置：`src/agents/definitions.ts:289+`

```
You are the relentless orchestrator of a multi-agent development system.

## RELENTLESS EXECUTION

You are BOUND to your task list. You do not stop. You do not quit. 
You do not take breaks. Work continues until EVERY task is COMPLETE.

## Your Core Duty
You coordinate specialized subagents to accomplish complex software 
engineering tasks. Abandoning work mid-task is not an option. 
If you stop without completing ALL tasks, you have failed.

## Available Subagents (19 Agents)
[详细的 Agent 列表和职责]
```

这个提示的关键特点：
- **强制持续性**："不停止、不放弃、不休息"
- **任务绑定**：必须完成所有任务
- **职责清晰**：每个 Agent 的具体职责

### 7.2 个别 Agent 提示

每个 Agent 都有自己的 Markdown 提示，动态加载：

```typescript
// agents/architect.md
# Architect Agent

You are the code architecture specialist...

## Your Responsibilities
- Analyze code structure
- Detect anti-patterns
- Propose architecture improvements
...
```

---

## 八、OMC 的配置系统

### 8.1 配置加载

位置：`src/config/loader.ts`

```typescript
function loadConfig(): PluginConfig {
  // 按优先级查找配置：
  // 1. .claude/claude-conf.jsonc (项目级)
  // 2. ~/.claude/claude-conf.jsonc (用户级)
  // 3. 默认配置
  
  const projectConfig = loadJsonc(projectConfigPath);
  const userConfig = loadJsonc(userConfigPath);
  
  // 深度合并
  return deepMerge(DEFAULT_CONFIG, userConfig, projectConfig);
}
```

### 8.2 配置结构

```typescript
interface PluginConfig {
  // Agent 配置
  agents?: {
    architect?: { model?: 'haiku' | 'sonnet' | 'opus' };
    executor?: { model?: 'haiku' | 'sonnet' | 'opus' };
    // ...
  };
  
  // 特性开关
  features?: {
    autoContextInjection?: boolean;    // 自动注入上下文
    continuationEnforcement?: boolean; // 强制任务完成
    lspTools?: boolean;               // LSP 工具
    astTools?: boolean;               // AST 工具
  };
  
  // 权限配置
  permissions?: {
    allowBash?: boolean;
    allowEdit?: boolean;
    allowWrite?: boolean;
  };
  
  // MCP 服务器配置
  mcpServers?: {
    exa?: { enabled?: boolean; apiKey?: string };
    context7?: { enabled?: boolean };
  };
  
  // 魔法关键词自定义
  magicKeywords?: Record<string, string>;
}
```

---

## 九、OMC 的工具系统

### 9.1 内置工具

OMC 通过 MCP 协议提供三类工具：

#### **LSP 工具（Language Server Protocol）**
- `lsp_hover` - 悬停信息
- `lsp_goto_definition` - 跳转定义
- `lsp_find_references` - 查找引用
- `lsp_diagnostics` - 代码诊断

#### **AST 工具（Abstract Syntax Tree）**
- `ast_grep_search` - AST 搜索
- `ast_grep_replace` - AST 替换

#### **Python REPL**
- 直接代码执行能力

### 9.2 状态管理工具

```typescript
// 读取工作块状态
readBoulderState()

// 写入工作块状态
writeBoulderState(newState)

// 追踪进度
getPlanProgress(planPath)

// 获取活跃计划
getActivePlanPath()
```

---

## 十、OMC 的 19 个 Agent 详细说明

### 10.1 Build/Analysis Lane（构建/分析车道）

| Agent | 模型 | 职责 | 何时使用 |
|-------|------|------|---------|
| **explore** | Haiku | 快速代码浏览，找关键文件 | 开始分析前 |
| **analyst** | Opus | 需求分析，找缺口和问题 | explore 之后 |
| **planner** | Opus | 创建可执行的工作计划 | 有明确目标时 |
| **architect** | Opus | 代码分析、架构验证、设计审查 | 需要深度分析时 |
| **executor** | Sonnet | 实际执行代码修改 | 准备好实施时 |

### 10.2 Verification Lane（验证车道）

| Agent | 模型 | 职责 | 何时使用 |
|-------|------|------|---------|
| **verifier** | Sonnet | 完成证据收集、测试验证 | 执行后需验证 |
| **debugger** | Sonnet | 根本原因分析、回归隔离 | 出现问题时 |

### 10.3 Review Lane（审查车道）

| Agent | 模型 | 职责 | 何时使用 |
|-------|------|------|---------|
| **security-reviewer** | Sonnet | 安全漏洞检测、OWASP 分析 | 安全审查时 |
| **code-reviewer** | Opus | 代码质量审查、最佳实践 | 代码审核时 |

### 10.4 Specialists（专家）

| Agent | 模型 | 职责 |
|-------|------|------|
| **test-engineer** | Sonnet | 测试策略、覆盖率分析 |
| **designer** | Sonnet | UI/UX 设计评审 |
| **writer** | Haiku | 文档编写、API 文档 |
| **git-master** | Sonnet | Git 操作、提交管理 |
| **qa-tester** | Sonnet | QA 测试、边界情况 |
| **scientist** | Sonnet | 数据分析、科学计算 |
| **tracer** | Sonnet | 证据追踪、因果分析 |
| **code-simplifier** | Opus | 代码简化、重构建议 |
| **critic** | Opus | 计划质量评审 |
| **document-specialist** | Sonnet | 文档查询、外部研究 |

---

## 十一、OMC 的作用（Use Cases）

### 11.1 场景 1：自动化完整功能开发

```
用户："/autopilot 构建用户认证系统"

OMC 自动流程：
1. explore → 分析现有架构
2. analyst → 确定认证需求
3. planner → 制定实施计划
4. architect → 验证架构合理性
5. executor → 编写代码
6. test-engineer → 编写测试
7. security-reviewer → 安全检查
8. code-reviewer → 代码审查
9. verifier → 最终验证

结果：完整的认证系统，生产就绪
```

### 11.2 场景 2：调试和修复

```
用户：编辑代码后，测试失败

OMC 修复流程：
1. explore → 定位失败位置
2. debugger → 根本原因分析
3. architect → 验证问题根源
4. executor → 修复代码
5. test-engineer → 更新测试
6. verifier → 验证修复成功

结果：自动修复，无需人工干预
```

### 11.3 场景 3：代码审查和优化

```
用户："/review 审查这个模块的代码"

OMC 审查流程：
1. code-reviewer → 代码质量审查
2. security-reviewer → 安全审查
3. code-simplifier → 简化建议
4. architect → 架构评价

结果：详细的审查报告 + 改进建议
```

### 11.4 场景 4：测试驱动开发

```
用户："/ultraqa 实现 checkout 流程"

OMC TDD 流程：
1. test-engineer → 生成测试用例
2. executor → 实现功能
3. qa-tester → 运行测试
4. 若失败，debugger → 修复
5. 循环直到所有测试通过

结果：高覆盖率的可靠代码
```

---

## 十二、OMC 的关键优势

### 12.1 架构优势

| 优势 | 说明 |
|------|------|
| **无学习曲线** | 不需要学习 Claude Code，直接使用自然语言 |
| **专业化分工** | 19 个专门 Agent，职责明确，高效协作 |
| **自动编排** | 无需手工选择 Agent，系统自动选择最佳工作流 |
| **状态追踪** | Boulder State 追踪进度，可恢复性强 |
| **模型智能路由** | Haiku/Sonnet/Opus 自动选择，成本和质量平衡 |
| **Hook 深度集成** | 与 Claude Code 原生集成，不破坏生态 |

### 12.2 功能优势

| 功能 | 说明 |
|------|------|
| **魔法关键词** | 一个词激活高级工作流（autopilot, ralph, ultrawork） |
| **自动验证** | 内置验证机制，确保工作质量 |
| **后台任务** | 支持并行执行多个长任务 |
| **上下文注入** | 自动收集 CLAUDE.md、AGENTS.md 等上下文 |
| **任务完成强制** | 确保任务不会因超时而中断 |
| **权限管理** | 细粒度的工具和文件访问控制 |

### 12.3 用户体验优势

| 优势 | 说明 |
|------|------|
| **零配置** | 安装即用，自动检测最优配置 |
| **命令简洁** | 一条命令启动复杂工作流 |
| **自适应** | 根据任务复杂度自动调整工作流 |
| **可中断** | 可随时中断，状态保存，可恢复 |
| **多语言** | 支持中文、日文、韩文、西班牙文等 |

---

## 十三、OMC 的核心设计哲学

### 13.1 "The Boulder Never Stops"

这是 OMC 的核心哲学，来自《西西弗斯的神话》隐喻：

```
You are BOUND to your task list. You do not stop. You do not quit. 
You do not take breaks. Work continues until EVERY task is COMPLETE.
```

**含义：**
- **持续驱动**：任务必须完成，不能中途放弃
- **自我修复**：遇到问题自动调用 debugger 修复
- **循环验证**：不断验证直到成功
- **不懈坚持**：这是 OMC 的DNA

### 13.2 "Zero Learning Curve"

```
Don't learn Claude Code. Just use OMC.
```

**设计理念：**
- **对用户友好**：用自然语言，不用学习命令
- **对专业人士友好**：自动化处理重复工作
- **对 AI 友好**：清晰的职责和上下文

### 13.3 "Separation of Concerns"

严格的 Agent 职责划分：

```
architect ≠ analyst ≠ planner ≠ critic ≠ executor
```

**好处：**
- 避免认知超载
- 提高执行效率
- 便于问题诊断
- 支持模型独立化

---

## 十四、OMC 的执行流程深入分析

### 14.1 执行层次

```
Level 1: User Input Layer
  ↓
  输入自然语言命令
  
Level 2: Keyword Detection Layer
  ↓
  detectMagicKeywords() 识别触发词
  
Level 3: Prompt Enhancement Layer
  ↓
  processPrompt() 增强提示词
  
Level 4: Session Creation Layer
  ↓
  createOmcSession() 准备编排环境
  
Level 5: Agent SDK Layer
  ↓
  Claude Agent SDK 执行
  
Level 6: Hook Interception Layer
  ↓
  Hook 系统截获关键事件
  
Level 7: Orchestration Layer
  ↓
  OMC 编排器决策 Agent 分配
  
Level 8: Agent Execution Layer
  ↓
  专门化 Agent 执行任务
  
Level 9: Tool Layer
  ↓
  LSP/AST/自定义工具执行
  
Level 10: Verification Layer
  ↓
  自动验证完成情况
  
Level 11: State Management Layer
  ↓
  Boulder State 保存进度
```

### 14.2 决策流程

```
Task Received
  ├─ Is it a keyword trigger? (autopilot, ralph, etc)
  │  └─ Yes → 使用对应技能
  │
  ├─ Is it a code change?
  │  └─ Yes → 使用 executor lane (explore → analyst → planner → executor)
  │
  ├─ Is it a bug/error?
  │  └─ Yes → 使用 debugger lane (debugger → analyzer → executor)
  │
  ├─ Is it a review request?
  │  └─ Yes → 使用 review lane (code-reviewer, security-reviewer)
  │
  └─ Is it a question/research?
     └─ Yes → 使用 document-specialist
```

---

## 十五、OMC 的应用场景总结

### 15.1 最佳适用场景

✅ **自动化功能开发** - 从规划到验证的完整工作流
✅ **自动化 Bug 修复** - 自动诊断和修复问题
✅ **代码审查** - 自动化质量、安全、最佳实践审查
✅ **测试驱动开发** - 自动生成和验证测试
✅ **性能优化** - 自动分析和优化瓶颈
✅ **文档生成** - 自动生成 API 文档、README
✅ **重构和简化** - 自动代码简化和优化
✅ **大规模并行任务** - ultrawork 模式下的高吞吐量执行

### 15.2 不适用或需要谨慎的场景

❌ **需要人工决策的任务** - 需要业务判断
❌ **高风险操作** - 如直接修改生产数据库
❌ **需要创意的任务** - 如创意文案写作
⚠️ **需要外部集成的任务** - 需要手工配置 API 密钥

---

## 总结

**Oh-My-ClaudeCode (OMC) 是一个智能多代理编排系统**，通过以下方式增强 Claude Code：

1. **19 个专门化 Agent** - 不同领域的专家
2. **魔法关键词系统** - 一个词激活复杂工作流
3. **Hook 深度集成** - 与 Claude Code 无缝集成
4. **Boulder State 追踪** - 工作进度持久化
5. **自动验证机制** - 确保质量
6. **模型智能路由** - 成本和质量平衡

**核心哲学：**
- 🪨 **"The Boulder Never Stops"** - 持续驱动，不懈坚持
- 🎯 **"Zero Learning Curve"** - 自然语言，无学习曲线
- 🔄 **"Separation of Concerns"** - 清晰的职责划分

**最终目标：** 让任何人都能通过自然语言命令，自动完成复杂的软件工程任务，从规划到验证到优化，全程自动化。

