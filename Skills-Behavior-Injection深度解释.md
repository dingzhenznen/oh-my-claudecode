# "Skills are Behavior Injections" — 深度解释与案例分析

## 核心概念：Behavior Injection vs Agent Swapping

### 问题陈述

**ARCHITECTURE.md 第 174-177 行说**：
```
Skills are behavior injections that modify how the orchestrator operates. 
Instead of swapping agents, skills add capabilities on top of existing agents.
```

这句话的关键是：**不替换 Agent，而是叠加能力**

---

## 一、什么是 Behavior Injection？

### 1.1 字面意思

```
Injection = 注入
Behavior = 行为

Behavior Injection = 将特定的行为注入到现有系统中，改变其运作方式
```

### 1.2 具体含义

**在 OMC 中**：

Skills 不是"替换"主 Agent，而是：
1. 保持主 Agent（如 executor）不变
2. 在其上层注入新的行为（如并行执行、自动重试、不停止直到完成）
3. 主 Agent 的执行被新行为"包装"或"增强"

---

## 二、两种错误的理解方式

### ❌ 错误理解 1：Agent Swapping（代理替换）

```
用户要求：并行执行代码
错误做法：
  ├─ 停用 executor Agent
  ├─ 启用 parallel-executor Agent（专门处理并行）
  └─ 结果：两个完全不同的 Agent

这叫 "Agent Swapping"（坏的设计）
```

**问题**：
- 创建太多 Agent，难以维护
- executor 和 parallel-executor 可能逻辑重复
- 用户需要记住什么时候用哪个 Agent

### ❌ 错误理解 2：Skill as Wrapper（完全替代）

```
错误做法：
  ├─ 用 Skill 替代整个 Agent 的功能
  ├─ Skill 自己处理所有逻辑
  └─ 结果：Skill = Agent

这也不对，因为 Skill 只是"行为增强"
```

---

## 三、正确理解：Behavior Injection

### ✅ 正确做法

```
用户要求：并行执行代码

正确做法：
  ├─ 保持 executor Agent 不变
  ├─ 注入 ultrawork Skill（添加并行能力）
  └─ 结果：executor 现在支持并行执行

这叫 "Behavior Injection"（好的设计）
```

**原理**：

```
executor Agent (原始)
  + ultrawork Skill (注入行为)
  ────────────────────────
  executor Agent with parallel execution (增强后)
```

---

## 四、用 Autopilot SKILL.md 来具体说明

### 4.1 Autopilot 是什么？

**定义**（SKILL.md 第 8-9 行）：
```
Autopilot takes a brief product idea and autonomously handles the full lifecycle: 
requirements analysis, technical design, planning, parallel implementation, 
QA cycling, and multi-perspective validation.
```

### 4.2 Autopilot 是一个 Behavior Injection

**重点**：Autopilot 不是一个新 Agent，而是一个**行为修改**

#### 不使用 Autopilot：

```
User: "build me a todo app"

Claude (主编排器) 默认行为：
  ├─ 理解请求
  ├─ 考虑是否需要规划
  ├─ 可能执行，但不保证完整的生命周期
  └─ 可能不跑测试，可能不做验证
```

#### 使用 Autopilot Skill：

```
User: "autopilot: build me a todo app"

注入 Autopilot Skill 后的行为：
  ├─ Phase 0: 需求扩展（分析 + 架构）
  ├─ Phase 1: 规划（创建计划 + 批评验证）
  ├─ Phase 2: 执行（Ralph + Ultrawork 并行）
  ├─ Phase 3: QA（循环直到测试通过）
  ├─ Phase 4: 验证（多角度审查）
  └─ Phase 5: 清理（删除临时文件）
```

**关键点**：编排器的**行为模式改变了**，但没有替换 Agent

### 4.3 Autopilot 的 6 阶段是"行为注入"

从 SKILL.md 第 39-74 行：

```
Phase 0 - Expansion (行为：详细需求扩展)
  ├─ 调用 analyst 和 architect
  └─ 输出规范到 .omc/autopilot/spec.md
  
Phase 1 - Planning (行为：严格的规划过程)
  ├─ Architect 创建计划
  ├─ Critic 验证计划
  └─ 输出计划到 .omc/plans/autopilot-impl.md
  
Phase 2 - Execution (行为：并行执行)
  ├─ Ralph + Ultrawork 结合
  ├─ 并行处理独立任务
  └─ 按复杂度选择 Executor (Haiku/Sonnet/Opus)
  
Phase 3 - QA (行为：自动 QA 循环)
  ├─ 构建、检查、测试、修复
  ├─ 循环最多 5 次
  └─ 如果 3 次重复同样错误，停止并报告
  
Phase 4 - Validation (行为：多角度验证)
  ├─ Architect：功能完整性
  ├─ Security-reviewer：漏洞检查
  ├─ Code-reviewer：质量审查
  └─ 所有审查必须通过
  
Phase 5 - Cleanup (行为：自动清理)
  └─ 删除状态文件
```

**这 6 个阶段是什么**？答案：**行为流程的注入**

编排器原本不做这 6 个阶段。当你激活 autopilot Skill，这个**行为模式被注入进来**。

---

## 五、对比不同的 Skills 来理解 Behavior Injection

### 5.1 Ralph vs Autopilot：不同的行为注入

#### Ralph SKILL.md 的行为：

```
Ralph (自我反思循环)
  ├─ 行为 1：持续循环直到完成
  ├─ 行为 2：跟踪 PRD 中的用户故事
  ├─ 行为 3：强制验证才能停止
  └─ 行为 4：跨迭代持久化进度
```

#### Autopilot 的行为：

```
Autopilot (完整生命周期)
  ├─ 行为 1：需求扩展
  ├─ 行为 2：规划验证
  ├─ 行为 3：并行执行
  ├─ 行为 4：自动 QA 循环
  └─ 行为 5：多角度验证
```

**关键观察**：
- Ralph 和 Autopilot **使用同样的 Agent**（executor, architect, verifier 等）
- 但它们**注入了不同的行为流程**
- 因此产生了**不同的执行模式**

### 5.2 CCG 的行为注入

从 CCG SKILL.md：

```
CCG (3 模型协作)
  注入的行为：
  ├─ 分解请求为两个提示
  ├─ 并行调用 Codex 和 Gemini
  ├─ 综合两个结果
  └─ 返回统一答案
```

**本质**：改变了编排器的**决策和调用方式**

---

## 六、架构层面：Agent vs Skill vs Behavior

### 6.1 三层架构

```
┌────────────────────────────┐
│ Behavior Layer (行为)       │
│ ─────────────────────      │
│ Skill 注入这一层            │
│ • 阶段顺序                 │
│ • 循环次数                 │
│ • 并行策略                 │
│ • 验证规则                 │
└────────────────────────────┘
         ↓ (指挥)
┌────────────────────────────┐
│ Agent Layer (执行)          │
│ ─────────────────────      │
│ 19 个专门化 Agent          │
│ • 每个做一件事             │
│ • 职责明确                 │
│ • 可组合                   │
└────────────────────────────┘
         ↓ (驱动)
┌────────────────────────────┐
│ Tool Layer (工具)           │
│ ─────────────────────      │
│ Read, Edit, Write, Bash... │
└────────────────────────────┘
```

**Skill 的作用**：在 Behavior Layer 修改执行流程

### 6.2 为什么这样设计？

**优点**：

```
✓ Agent 可以复用
  ├─ 同一个 executor 可以用不同的方式执行
  └─ 不需要为每种执行方式创建新 Agent

✓ Behavior 易于扩展
  ├─ 添加新的行为流程只需添加新 Skill
  └─ 不需要修改 Agent 代码

✓ 组合灵活
  ├─ [Execution] + [0-N Enhancements] + [Optional Guarantee]
  ├─ default + ultrawork + git-master + ralph
  └─ 每个 Skill 添加一种能力
```

---

## 七、工程示例：behavior injection 的具体流程

### 7.1 没有 Skill：默认行为

```
User: "implement a login feature"

默认编排流程：
1. Claude 理解请求
2. 可能委派给 executor Agent
3. executor 修改代码
4. 完成
（可能缺少规划、测试、验证）
```

### 7.2 注入 autopilot Skill

```
User: "autopilot: build me a complete authentication system"

注入 autopilot 行为后：
1. Phase 0 - 需求扩展
   ├─ analyst Agent 分析需求
   ├─ architect Agent 创建规范
   └─ 生成 .omc/autopilot/spec.md

2. Phase 1 - 规划
   ├─ architect Agent 创建计划
   ├─ critic Agent 批评计划
   └─ 生成 .omc/plans/autopilot-impl.md

3. Phase 2 - 执行（Ralph + Ultrawork）
   ├─ 并行启动多个 executor Agent
   ├─ 按复杂度选择模型
   └─ 修改代码

4. Phase 3 - QA 循环
   ├─ 运行测试
   ├─ 若失败，修复代码
   ├─ 重复最多 5 次
   └─ 若 3 次同样错误，停止报告

5. Phase 4 - 验证
   ├─ architect 验证功能
   ├─ security-reviewer 检查安全
   ├─ code-reviewer 审查代码
   └─ 所有通过才认为完成

6. Phase 5 - 清理
   └─ 删除临时状态文件
```

**关键观察**：

- Agent 还是那些 Agent（analyst, architect, executor, critic 等）
- 但是**执行流程完全不同**
- 这个不同的流程是由 Autopilot Skill **注入的**

### 7.3 注入 ralph Skill

```
User: "ralph: implement multi-factor authentication"

注入 ralph 行为后（不同的行为注入）：
1. 初始化 PRD，定义用户故事
2. 循环：
   ├─ 选择第一个未完成的故事
   ├─ 委派给 executor
   ├─ 验证故事是否满足
   ├─ 若未满足，标记失败，重试
   ├─ 若满足，标记通过
   └─ 继续下一个故事
3. 所有故事通过后，强制验证（由 critic 或 architect）
4. 只有验证通过才停止
```

**对比**：
- Autopilot：6 个阶段的严格流程
- Ralph：循环故事验证的持久流程
- 用的是**同样的 Agent**，但**行为流程不同**

---

## 八、核心洞察：为什么叫"Injection"？

### 8.1 "Injection" 的含义

在编程中，Injection 通常指：

```
不修改原有代码，而是在运行时注入新的行为
```

**例子**：
```
原始函数：
  function execute(task) {
    return agent.run(task);  // 执行任务
  }

注入行为后（就像 Skill 做的）：
  function executeWithAutopilot(task) {
    spec = phase0_expand(task);        // 注入：扩展
    plan = phase1_plan(spec);          // 注入：规划
    code = phase2_execute(plan);       // 注入：执行（可能并行）
    verify = phase3_qa(code);          // 注入：测试
    final = phase4_validate(verify);   // 注入：验证
    return final;
  }
```

**关键**：底层的 Agent 代码没变，但是**行为被改变了**

### 8.2 OMC 中的 Injection

```
OMC Hook 系统检测到 "autopilot" 关键词
  ↓
Hook 从 SKILL.md 加载 Skill 定义
  ↓
Skill 的行为流程被注入到编排器中
  ↓
编排器按照新的行为流程执行
  ↓
结果：同样的 Agent，不同的执行模式
```

---

## 九、技术实现：Skill 如何注入行为？

### 9.1 从源码层面

Hook 系统（第 320-425 行）在 `UserPromptSubmit` 事件捕获 Skill：

```
keyword-detector Hook
  ├─ 检测 "autopilot" 关键词
  ├─ 查找 skills/autopilot/SKILL.md
  ├─ 读取 Skill 的 <Purpose>, <Steps>, <Execution_Policy>
  └─ 注入到编排器
```

### 9.2 Autopilot 的行为注入点

**SKILL.md 第 39-74 行定义的 6 个 Phase**：

```
这些 Phase 就是"行为注入"
每个 Phase 说明了应该做什么、怎样做、何时停止
```

**SKILL.md 第 31-37 行的 Execution_Policy**：

```
- Each phase must complete before the next begins
- Parallel execution is used within phases where possible
- QA cycles repeat up to 5 times
- Validation requires approval from all reviewers
- Cancel with /oh-my-claudecode:cancel at any time

这些策略改变了编排器的决策
```

---

## 十、Skills 之间的组合：多层 Injection

### 10.1 组合公式回顾

```
[Execution Skill] + [0-N Enhancements] + [Optional Guarantee]
```

**例**：
```
default + ultrawork + git-master + ralph
  │         │          │           │
  │         │          │           └─ Guarantee: 不停止直到完成
  │         │          └─ Enhancement: 原子提交
  │         └─ Enhancement: 并行执行
  └─ Execution: 基础执行
```

### 10.2 多层 Injection 的含义

```
Phase 1: default 注入基础行为
  └─ "执行任务"

Phase 2: ultrawork 注入增强行为
  └─ + "并行执行"

Phase 3: git-master 注入专项行为
  └─ + "创建原子提交"

Phase 4: ralph 注入保证行为
  └─ + "直到验证才停止"

结果：
  多层行为注入 = 复杂的执行流程
```

---

## 十一、总结：Behavior Injection 的精髓

### 最核心的概念

```
Skills are behavior injections
 ↓
Skills 改变执行流程（不改变 Agent 本身）
 ↓
同样的 Agent，不同的 Skill，产生不同的行为
 ↓
这就是"注入"的含义
```

### vs Agent Swapping 的对比

| 特性 | Behavior Injection (✓ OMC) | Agent Swapping (✗) |
|------|---------------------------|-------------------|
| 加新功能 | 注入新的行为流程 | 创建新的 Agent |
| Agent 复用 | ✓ 同一 Agent 多个 Skill | ✗ 每个功能新 Agent |
| 维护难度 | 低（Skill 独立） | 高（Agent 重复逻辑） |
| 灵活性 | 高（Skill 可组合） | 低（Agent 固定） |
| 学习曲线 | 低（只需理解行为） | 高（需学习每个 Agent） |

---

## 十二、从 Autopilot 看"不替换，而是叠加"

### 错误理解：Skill 替换 Agent

```
❌ "Autopilot 是一个 Agent"
"Autopilot 替换 executor，自己处理所有逻辑"
```

### 正确理解：Skill 增强 Agent

```
✓ "Autopilot 是一个 Skill，不是 Agent"
"Autopilot 改变编排器的行为流程"
"实际执行还是用 executor, architect, critic 等 Agent"
"Autopilot 只是指导它们怎样执行"
```

### 证据：看 Autopilot 的代理调用

**SKILL.md 第 40-70 行**：

```
Phase 0: analyst (Agent) 和 architect (Agent) 分析
Phase 1: architect (Agent) 和 critic (Agent) 规划
Phase 2: executor (Agent) 执行
Phase 3: 运行测试（不调用新 Agent）
Phase 4: architect (Agent), security-reviewer (Agent), code-reviewer (Agent) 验证
Phase 5: 清理
```

**关键**：Autopilot 使用的 Agent 都是**原有的 19 个 Agent**

Autopilot 本身**不做任何真实工作**，只是指挥这些 Agent 按特定流程执行。

这正是"**不替换 Agent，而是叠加能力**"的含义。

---

## 总结表格

| 概念 | 说明 | 例子 |
|------|------|------|
| **Agent** | 执行单元，做真实工作 | executor, architect |
| **Skill** | 行为流程，指导 Agent 怎样执行 | autopilot, ralph, ultrawork |
| **Behavior Injection** | 在运行时改变执行流程 | 激活 autopilot，执行流程从简单变复杂 |
| **组合** | 多个 Skill 叠加 | default + ultrawork + git-master + ralph |

**最后一句话**：

> Skills are behavior injections — 它们改变了编排器**如何协调 Agent**，而不是替换 Agent 本身。Agent 还是那 19 个 Agent，但在不同的 Skill 下，它们的**执行流程、并行度、验证规则完全不同**。这就是"注入"的力量。

