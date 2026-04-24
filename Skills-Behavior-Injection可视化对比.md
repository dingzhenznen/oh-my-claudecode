# Skill 作为 Behavior Injection 的可视化对比

## 一、核心概念可视化

### 1.1 错误 vs 正确的设计理念

#### ❌ 错误方式：Agent Swapping

```
User: "我需要并行执行任务"

错误做法：
┌──────────────────────────────┐
│ 注册新的 Agent：              │
│ parallel-executor             │
│ multi-task-executor          │
│ async-executor               │
│ batch-executor               │
│ ...                          │
└──────────────────────────────┘
      ↓
问题：
├─ Agent 数量爆炸（100+ 个？）
├─ 每个 Agent 可能有重复逻辑
├─ 维护者不知道什么时候用哪个
└─ 用户需要记住所有 Agent 名字
```

#### ✅ 正确方式：Behavior Injection

```
User: "我需要并行执行任务"

正确做法：
┌──────────────────────────────┐
│ 保持 19 个 Agent 不变         │
│ 注入 ultrawork Skill         │
│                              │
│ [ultrawork 修改行为]         │
│ executor 现在支持并行        │
└──────────────────────────────┘
      ↓
优点：
├─ Agent 数量稳定（就 19 个）
├─ 无代码重复
├─ 用户只需记住 Skill
└─ 可组合多个 Skill
```

---

## 二、执行流程对比

### 2.1 不使用任何 Skill

```
User: "build a todo app"

默认编排流程：
┌─────────────────────────────────────────┐
│ 1. 理解请求                              │
│    executor → 写代码                     │
│                                          │
│ 2. 完成                                  │
│                                          │
│ （可能缺少规划、测试、验证）              │
└─────────────────────────────────────────┘

执行路径：
user request
    ↓
executor (实施)
    ↓
done
```

### 2.2 注入 Autopilot Skill

```
User: "autopilot: build me a todo app"

Autopilot 注入的行为流程：
┌─────────────────────────────────────────────────────┐
│ Phase 0: 需求扩展                                    │
│  ├─ analyst 分析需求                                │
│  └─ architect 创建规范                              │
│      ↓                                             │
│ Phase 1: 规划                                        │
│  ├─ architect 创建计划                              │
│  └─ critic 批评计划                                 │
│      ↓                                             │
│ Phase 2: 执行（Ralph + Ultrawork）                 │
│  ├─ 并行启动 N 个 executor                          │
│  └─ 按复杂度选择模型 (Haiku/Sonnet/Opus)           │
│      ↓                                             │
│ Phase 3: QA 循环                                     │
│  ├─ 运行测试                                        │
│  ├─ 若失败，修复                                    │
│  └─ 循环最多 5 次                                   │
│      ↓                                             │
│ Phase 4: 验证（多角度）                             │
│  ├─ architect 验证功能                              │
│  ├─ security-reviewer 检查安全                      │
│  └─ code-reviewer 审查代码                          │
│      ↓                                             │
│ Phase 5: 清理                                        │
│  └─ 删除状态文件                                    │
│      ↓                                             │
│ All done and verified                              │
└─────────────────────────────────────────────────────┘

执行路径：
user request (with autopilot keyword)
    ↓
[Autopilot Skill 注入行为]
    ↓
analyzer + architect + executor + critic + 
security-reviewer + code-reviewer + verifier
    ↓
verified result
```

### 2.3 注入 Ralph Skill（不同的行为）

```
User: "ralph: implement feature X"

Ralph 注入的行为流程：
┌──────────────────────────────────┐
│ 初始化 PRD，定义用户故事         │
│                                  │
│ 循环：                           │
│ ├─ 选择未完成的故事              │
│ ├─ 委派给 executor              │
│ ├─ 验证是否满足                  │
│ ├─ 若未满足，标记失败，重试      │
│ ├─ 若满足，标记通过              │
│ └─ 直到所有故事通过              │
│                                  │
│ 强制 Verification               │
│ 只有通过才停止                   │
└──────────────────────────────────┘

关键特点：
├─ 持久化循环（不轻易放弃）
├─ PRD 驱动（每个故事都要验证）
└─ 强制完成（不完成就不停止）
```

---

## 三、同一个 Agent 在不同 Skill 下的执行方式

### 3.1 Executor Agent 在不同 Skill 下

```
Executor Agent：代码实现专家

┌────────────────────────────────────────────┐
│ 没有 Skill（默认）                         │
├────────────────────────────────────────────┤
│ 执行方式：                                 │
│ ├─ 序列执行（一个任务接一个）             │
│ ├─ 可能不完整（任务可能遗漏）             │
│ └─ 时间：长（无并行）                     │
└────────────────────────────────────────────┘

          ↓ (同一个 Agent)

┌────────────────────────────────────────────┐
│ + Ultrawork Skill（并行）                  │
├────────────────────────────────────────────┤
│ 执行方式：                                 │
│ ├─ 并行执行（同时处理多个任务）           │
│ ├─ 独立任务不相互阻塞                     │
│ └─ 时间：短（有并行）                     │
└────────────────────────────────────────────┘

          ↓ (同一个 Agent)

┌────────────────────────────────────────────┐
│ + Ralph Skill（循环直到完成）              │
├────────────────────────────────────────────┤
│ 执行方式：                                 │
│ ├─ 循环执行（直到验证通过）               │
│ ├─ 失败则重试，不轻易放弃                 │
│ └─ 强制完成（不完成不停止）               │
└────────────────────────────────────────────┘

          ↓ (同一个 Agent)

┌────────────────────────────────────────────┐
│ + Autopilot Skill（完整生命周期）          │
├────────────────────────────────────────────┤
│ 执行方式：                                 │
│ ├─ 在 6 个 Phase 中被调用                 │
│ ├─ 在 Phase 2（执行）中工作               │
│ └─ 被严格的生命周期框架指导               │
└────────────────────────────────────────────┘
```

**结论**：
```
同一个 executor Agent
├─ 默认：序列、可能不完整
├─ + ultrawork：并行、高效
├─ + ralph：循环、必完成
└─ + autopilot：生命周期、全验证

这就是 "Behavior Injection"
```

---

## 四、Skill 的三层组合模型

### 4.1 技能组合的可视化

```
Execution Layer (执行层 - 必需)
┌─────────────────────────────┐
│ default (基础执行)           │
│ build (构建任务)            │
│ orchestrate (协调)          │
└─────────────────────────────┘

         ↑ (叠加)

Enhancement Layer (增强层 - 0-N 个)
┌─────────────────────────────┐
│ ultrawork (并行)            │
│ git-master (原子提交)       │
│ frontend-ui-ux (UI 增强)    │
│ ... (其他增强)              │
└─────────────────────────────┘

         ↑ (叠加)

Guarantee Layer (保证层 - 可选)
┌─────────────────────────────┐
│ ralph (不停止直到完成)      │
│ (其他保证)                  │
└─────────────────────────────┘
```

### 4.2 组合示例

#### 组合 1：Simple

```
default
│
└─→ 基础执行，无增强
```

#### 组合 2：Parallel

```
default
├─ + ultrawork
│
└─→ 基础执行 + 并行处理
```

#### 组合 3：Parallel with Git

```
default
├─ + ultrawork
├─ + git-master
│
└─→ 基础执行 + 并行处理 + 原子提交
```

#### 组合 4：Full Autopilot

```
default (在 autopilot 的 Phase 2 中使用)
├─ + ultrawork (Phase 2 中的并行)
├─ + git-master (Phase 5 后的提交)
├─ + ralph (贯穿整个过程的保证)
│
└─→ Autopilot 的 6 阶段完整流程
```

---

## 五、Autopilot 具体的行为注入示例

### 5.1 Autopilot 的 6 个 Phase

每个 Phase 都是**行为流程的一部分**：

```
Phase 0: Expansion (行为：自动扩展需求)
─────────────────────────────────────────────
默认行为：不扩展，直接实施
注入后：自动调用 analyst 和 architect 分析需求

Phase 1: Planning (行为：强制规划和批评)
─────────────────────────────────────────────
默认行为：可能跳过规划
注入后：必须经过 architect 和 critic 验证

Phase 2: Execution (行为：Ralph + Ultrawork 结合)
─────────────────────────────────────────────────
默认行为：序列执行
注入后：并行执行，直到所有任务完成

Phase 3: QA (行为：自动 QA 循环)
─────────────────────────────────────────────
默认行为：可能无测试
注入后：自动运行测试，循环修复，最多 5 次

Phase 4: Validation (行为：多角度验证)
─────────────────────────────────────────────
默认行为：可能无验证
注入后：必须通过 architect + security-reviewer + code-reviewer

Phase 5: Cleanup (行为：自动清理)
─────────────────────────────────────────────
默认行为：无清理
注入后：删除所有临时文件
```

### 5.2 从 SKILL.md 看注入的具体行为

**SKILL.md 第 31-37 行：Execution Policy**

```
- Each phase must complete before the next begins
  ↓ 注入的行为：严格的阶段顺序

- Parallel execution is used within phases where possible
  ↓ 注入的行为：Phase 2 和 Phase 4 并行

- QA cycles repeat up to 5 times; if the same error persists 3 times, stop
  ↓ 注入的行为：自动重试，但有上限

- Validation requires approval from all reviewers; rejected items get fixed and re-validated
  ↓ 注入的行为：强制所有审查人通过

- Cancel with /oh-my-claudecode:cancel at any time; progress is preserved
  ↓ 注入的行为：支持中断和恢复
```

这些都是**编排流程的改变**，而不是 Agent 本身的改变。

---

## 六、核心对比：Injection vs Swapping

### 6.1 核心差异表

| 维度 | Behavior Injection (✓) | Agent Swapping (✗) |
|-----|----------------------|-------------------|
| **新增功能** | 注入新行为流程 | 创建新 Agent |
| **Agent 复用** | ✓ 19 个 Agent 服务所有 Skill | ✗ 每个 Skill 新 Agent |
| **代码重复** | ✗ 无 | ✓ 有 |
| **维护者角度** | 改 Skill 文件 | 改 Agent 代码 |
| **用户角度** | 选择 Skill | 选择 Agent |
| **扩展性** | 高（Skill 易添加） | 低（Agent 难扩展） |
| **学习曲线** | 低 | 高 |

### 6.2 现实代码组织

#### ❌ 错误的组织（如果用 Swapping）

```
agents/
├─ executor.ts
├─ parallel-executor.ts
├─ loop-executor.ts
├─ auto-executor.ts
├─ batch-executor.ts
├─ ... (100+ 个 executor 变种)
└─ 维护噩梦
```

#### ✅ 正确的组织（用 Injection）

```
agents/
├─ executor.ts (就这一个)

skills/
├─ default/SKILL.md
├─ ultrawork/SKILL.md
├─ ralph/SKILL.md
├─ autopilot/SKILL.md
├─ ... (31 个 Skill)
```

---

## 七、从"注入"的隐喻理解

### 7.1 现实中的"注入"

```
疫苗注射：
├─ 你的免疫系统不变
├─ 注射疫苗，改变免疫系统的"行为"
├─ 现在你的免疫系统会识别特定病原体
└─ 结果：新的防御能力

OMC 中的"注入"：
├─ Agent 的能力不变
├─ 注入 Skill，改变编排流程
├─ 现在编排器按新的方式协调 Agent
└─ 结果：新的执行流程
```

### 7.2 代码中的"注入"

```
原始系统：
  function execute(task) {
    return agent.run(task);  // 简单执行
  }

注入"并行"行为：
  function executeWithParallel(task) {
    for each independent_subtask in task {
      execute_in_parallel(subtask);  // 注入的行为
    }
    wait_for_all();
  }

注入"循环"行为：
  function executeWithLoop(task) {
    while (not_verified) {
      execute(task);                // 原始执行
      if (failed) retry();          // 注入的行为
    }
  }

注入"Autopilot"行为：
  function executeWithAutopilot(task) {
    phase0_expand(task);           // 注入
    phase1_plan(expanded);         // 注入
    phase2_execute(plan);          // 原始
    phase3_qa(code);               // 注入
    phase4_validate(qa_result);    // 注入
  }
```

---

## 八、总结：Behavior Injection 的力量

### 核心对比

```
Agent Swapping:
"需要新功能" → "创建新 Agent" → "学习新 Agent" → "维护新代码"

Behavior Injection:
"需要新功能" → "编写新 Skill" → "注入行为" → "Agent 复用"
```

### 为什么是 Injection？

```
✓ 低耦合
  ├─ Agent 代码不需要改
  └─ Skill 独立修改

✓ 高复用
  ├─ 同一个 Agent，多种执行方式
  └─ 19 个 Agent 支持 31 个 Skill

✓ 易扩展
  ├─ 添加新 Skill 只需新建文件
  └─ 不涉及 Agent 改动

✓ 易理解
  ├─ 用户选择 Skill，不用记住所有 Agent
  └─ 每个 Skill 独立文档化
```

### 最后的理解

```
Skills are behavior injections
         ↓
Skills 不创建新 Agent
Skills 改变编排流程
         ↓
同样的 Agent
不同的 Skill
不同的执行方式
         ↓
这就是 OMC 的优雅之处
```

