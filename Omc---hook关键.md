
› › 根据'/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/docs/ARCHITECTURE.md'
    结合'/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/docs/CLAUDE.md'，
    帮我理解'/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/skills/autopilot/SKILL.md'
    autopilot 这个skill 如果工作的


• 我会先读取相关的架构文档和 autopilot skill 文件，按它们之间的关系梳理工作机制。

• Explored
  └ Read SKILL.md (using-superpowers skill), SKILL.md, CLAUDE.md, ARCHITECTURE.md

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 我已经看到三份文件的核心关系：CLAUDE.md 是运行时总规则，ARCHITECTURE.md 解释四个系统如何协作，autopilot/
  SKILL.md 是具体流水线定义。为了把“它如何工作”讲完整，我再补一下架构文档后半部分里关于 hooks/state 的细节。

• Explored
  └ Read ARCHITECTURE.md
    Search */hooks/* in oh-my-claudecode

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• autopilot 可以理解成 oh-my-claudecode 里“从一句产品需求到已验证代码”的最高级自动执行流水线。它不是一个单独
  agent，而是一个 skill 编排器：被触发后，它按固定阶段调用不同 agents、组合其他 skills、写入 .omc/ 状态，并通过
  hooks 保持可恢复和可验证。

  整体工作链路

  当用户输入类似：

  autopilot build me a REST API with authentication
  build me a CLI habit tracker
  I want a todo app using TypeScript

  CLAUDE.md 和 hook 系统会检测 magic keyword：

  "autopilot" → autopilot
  "build me" → autopilot
  "I want a" → autopilot

  然后激活 oh-my-claudecode/skills/autopilot/SKILL.md:1 里的执行协议。

  它的 6 个阶段

  1. Phase 0 - Expansion
     把用户的简短想法扩展成详细 spec。

     如果输入很模糊，比如“build me something cool”，它不会直接硬做，而是建议先走 /deep-interview。
     如果已经存在 .omc/specs/deep-interview-*.md，就直接复用这个 spec。
     如果已经存在 ralplan 共识计划，则连 Phase 0 和 Phase 1 都跳过。

     输出：

     .omc/autopilot/spec.md

  2. Phase 1 - Planning
     根据 spec 生成实现计划。

     通常由：

     Architect → 创建计划
     Critic → 审查计划

     输出：

     .omc/plans/autopilot-impl.md

  3. Phase 2 - Execution
     真正写代码。

     这里会结合 ralph + ultrawork 的思想：
      - ralph：持续推进直到完成
      - ultrawork：能并行的任务并行做
      - executor agents：按复杂度使用 haiku / sonnet / opus

     简单任务给 Haiku，标准实现给 Sonnet，复杂任务给 Opus。
  4. Phase 3 - QA
     构建、lint、测试、修复，最多循环 5 次。

     如果同一个错误连续出现 3 次，它会停止并报告这是“根本性问题”，需要人介入，而不是无限重试。
  5. Phase 4 - Validation
     多角度并行验证：

     Architect          → 功能完整性
     Security-reviewer  → 安全问题
     Code-reviewer      → 代码质量

     三方都必须通过。任何一个 reject，都要修复后重新验证。最多 3 轮。

  6. Phase 5 - Cleanup
     成功后清理运行状态：

     .omc/state/autopilot-state.json
     .omc/state/ralph-state.json
     .omc/state/ultrawork-state.json
     .omc/state/ultraqa-state.json

     然后执行 cancel 做干净退出。

  它和架构文档的关系

  oh-my-claudecode/docs/ARCHITECTURE.md:1 说 OMC 有四个核心系统：

  Hooks → Skills → Agents → State

  autopilot 正好把这四个都用上了：

  用户输入
    ↓
  Hooks 检测 autopilot / build me / I want a
    ↓
  Skills 激活 autopilot 行为
    ↓
  Agents 执行 analyst / architect / critic / executor / verifier 等角色
    ↓
  State 写入 .omc/，支持恢复、追踪、清理

  所以 autopilot 的本质是：

  一个端到端执行模式，不是一个写代码 agent。
  它负责决定什么时候分析、什么时候规划、什么时候并行实现、什么时候 QA、什么时候验证。

  什么时候适合用

  适合：

  autopilot build me a bookstore inventory REST API using TypeScript
  build me a habit tracking CLI with streak counting
  I want a dashboard for monitoring Kubernetes deployments

  不适合：

  fix the login bug
  explain this file
  review this plan
  what are some caching approaches?

  这些更适合直接问、用 ralph、用 plan，或者委托单个 executor/reviewer。

  最重要的一句话

  autopilot 是 OMC 的“全自动产品开发流水线”：它把一个简短需求变成 spec、计划、实现、测试、审查和清理，全程依赖
  hooks 激活、skills 注入行为、agents 执行专业任务、.omc/ 保存状态。

───────────────────────────────────────────────────────────────────────────────────────────────────────────────


› claude 和 skill 中都没有  Architect ，Critic等agent 的定义，skill 是如何使用 他们的


• 我会查一下 OMC 里 agents 的实际定义位置和安装/引用方式，因为 SKILL.md 只描述“调用谁”，agent 本体通常不写在
  skill 文件里。

• Explored
  └ Search agents in oh-my-claudecode
    Read SKILL.md
    Search name: (architect|critic|executor|planner|analyst|security-reviewer|code-
    reviewer)|subagent_type=\"oh-my-
           claudecode:(architect|critic|executor|planner|analyst)|oh-my-claudecode:architect|architect in oh-
           my-
           claudecode

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 查到了：agent 定义确实不在 CLAUDE.md 或 skill 文件里，而是在仓库的 agents/*.md 和 src/agents/* 里；skill 通过
  Claude Code 的 Task(subagent_type=...) 引用这些已注册 agent。下面我再看一下注册表和安装逻辑，确认它们是怎么进
  入 Claude Code 的。

• Explored
  └ Read critic.md, architect.md, definitions.ts, index.ts

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 我需要最后确认 agents/*.md 是如何被安装到 Claude 的 agent 目录，和 oh-my-claudecode: 前缀如何处理。这样能回
  答“skill 怎么使用它们”而不是只说“文件在这里”。

• Explored
  └ Read definitions.ts, utils.ts
    Search agents/|\.claude/agents|subagent_type|oh-my-claudecode: in oh-my-claudecode

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 对，你的理解是对的：CLAUDE.md 和 autopilot/SKILL.md 里确实没有完整定义 Architect、Critic 这些 agent。它们是通
  过 agent 注册系统 被 Claude Code 识别的，skill 只是“引用”它们。

  核心链路是：

  autopilot skill
    ↓
  提示 orchestrator 使用 Task(subagent_type="oh-my-claudecode:architect", ...)
    ↓
  Claude Code / OMC 根据 subagent_type 查 agent 注册表
    ↓
  加载 agents/architect.md 里的 prompt + model + tool 限制
    ↓
  启动 architect 子代理

  实际定义在这里：

  oh-my-claudecode/agents/architect.md
  oh-my-claudecode/agents/critic.md
  oh-my-claudecode/agents/executor.md
  ...

  例如 oh-my-claudecode/agents/architect.md:1 里有：

  name: architect
  description: Strategic Architecture & Debugging Advisor (Opus, READ-ONLY)
  model: opus
  disallowedTools: Write, Edit

  oh-my-claudecode/agents/critic.md:1 里也类似：

  name: critic
  model: opus
  disallowedTools: Write, Edit

  所以 agent 的完整行为定义在 agents/*.md，不是在 skill 里。

  src/agents/definitions.ts 则是注册表，把这些 agent 组织起来：

  const agents = {
    explore: exploreAgent,
    analyst: analystAgent,
    planner: plannerAgent,
    architect: architectAgent,
    debugger: debuggerAgent,
    executor: executorAgent,
    verifier: verifierAgent,
    'security-reviewer': securityReviewerAgent,
    'code-reviewer': codeReviewerAgent,
    critic: criticAgent,
    ...
  }

  对应位置是 oh-my-claudecode/src/agents/definitions.ts:198。

  loadAgentPrompt() 会从 agents/{agentName}.md 读取 prompt：

  const agentPath = join(agentsDir, `${agentName}.md`);
  const content = readFileSync(agentPath, 'utf-8');
  return stripFrontmatter(content);

  位置在 oh-my-claudecode/src/agents/utils.ts:82。

  所以 skill 使用 agent 的方式不是：

  skill 内部包含 Architect 定义

  而是：

  skill 说：请调用 architect
  OMC/Claude Code 运行时去 agent 注册表找 architect 的定义

  oh-my-claudecode: 前缀只是命名空间。比如：

  oh-my-claudecode:architect

  会被 OMC 的 delegation 逻辑规范化成 canonical agent：

  architect

  然后再根据注册表补充默认模型、prompt、工具限制等。

  总结一句：

  autopilot/SKILL.md 是“流程说明书”，agents/*.md 是“角色说明书”，src/agents/definitions.ts 是“角色注册表”。
  autopilot 通过 Task(subagent_type="oh-my-claudecode:architect") 这类委派语句，让 Claude Code 启动已经注册好的
  agent。

─ Worked for 1m 04s ───────────────────────────────────────────────────────────────────────────────────────────


› '/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/skills/autopilot/SKILL.md'
  中**Phase 1 - Planning**: Create an implementation plan from the spec
     - **If ralplan consensus plan exists**: Skip — already done in the 3-stage pipeline
     - Architect (Opus): Create plan (direct mode, no interview)
     - Critic (Opus): Validate plan
     - Output: `.omc/plans/autopilot-impl.md`
  并没有使用  Task(）来委派Architect Critic，Critic 是如何工作的


• 我会确认 autopilot 是否还有命令模板或 hook 运行时代码；如果没有，那这里的 Architect (Opus) 就是给主
  orchestrator 的自然语言调度指令，而不是程序调用。

• Explored
  └ Search autopilot|Phase 1|autopilot-impl|Architect \(Opus\)|Critic \(Opus\) in oh-my-claudecode
    Read delegation-enforcer.ts, keyword-detector.mjs

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 搜索结果显示除了 skill 文本，还有 src/hooks/autopilot 的运行时类型和 adapter，但 SKILL.md 的 Phase 1 本身没有
  硬编码 Task。我要看这个 autopilot hook/adapter 是否实际生成了 agent 调用，还是只管理状态。

• Explored
  └ Read ralplan-adapter.ts
    List autopilot
    Read types.ts
    Search critic|architect|Task\(|subagent_type|planning|Phase in autopilot

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 关键点已经明确：SKILL.md 是简化版/声明式说明，真正给模型看的 Phase 1 执行提示由 src/hooks/autopilot/
  prompts.ts 生成，那里明确包含 Task(...critic...)。

• Explored
  └ Read pipeline-types.ts, prompts.ts, enforcement.ts

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 你指出的是 SKILL.md 里的简写描述；真正运行时会有更具体的 prompt 生成逻辑。

  在源码里，Phase 1 的直接规划 prompt 位于：

  oh-my-claudecode/src/hooks/autopilot/prompts.ts:111

  里面明确生成了两个 Task(...)：

  Task(
    subagent_type="oh-my-claudecode:architect",
    model="opus",
    prompt="CREATE IMPLEMENTATION PLAN ..."
  )

  然后：

  Task(
    subagent_type="oh-my-claudecode:critic",
    model="opus",
    prompt="REVIEW IMPLEMENTATION PLAN ..."
  )

  也就是说：

  SKILL.md 里这段：

  Architect (Opus): Create plan
  Critic (Opus): Validate plan

  不是最终完整执行代码，只是 skill 的高层说明。真正让 Critic 工作的是运行时生成的 Phase prompt，里面包含
  Task(subagent_type="oh-my-claudecode:critic", model="opus", ...)。

  运行链路大概是：

  autopilot skill 被激活
    ↓
  autopilot state 进入 planning phase
    ↓
  enforcement.ts 调用 getPhasePrompt(...)
    ↓
  getPhasePrompt("planning") 调用 getDirectPlanningPrompt(...)
    ↓
  prompts.ts 生成包含 Architect Task + Critic Task 的具体指令
    ↓
  Claude 主 orchestrator 看到 Task(...)，启动 critic 子代理

  关键位置：

  - oh-my-claudecode/src/hooks/autopilot/enforcement.ts:336：根据当前 phase 调 getPhasePrompt
  - oh-my-claudecode/src/hooks/autopilot/prompts.ts:133：Architect Task
  - oh-my-claudecode/src/hooks/autopilot/prompts.ts:171：Critic Task

  另外，当前 pipeline 默认配置是：

  planning: "ralplan"

  见 oh-my-claudecode/src/hooks/autopilot/pipeline-types.ts:85。

  如果走 planning: "ralplan"，它会通过 ralplan 工作流做 Planner / Architect / Critic 共识规划；如果走 planning:
  "direct"，才是 Architect 创建计划 + Critic 审查计划 这套直接规划逻辑。

  所以准确说：

  SKILL.md 本身没有直接调用 Critic，但 OMC 的 autopilot runtime 会把 Phase 1 展开成包含 Task(subagent_type="oh-
  my-claudecode:critic") 的执行 prompt，Critic 就是在这个阶段被 Claude Code 作为子代理启动的。

─ Worked for 1m 01s ───────────────────────────────────────────────────────────────────────────────────────────


› getDirectPlanningPrompt 是在什么时候调用的


• 我会沿着 getDirectPlanningPrompt 的调用链查到入口条件，重点看 getPhasePrompt 和 pipeline adapter 两条路径。

• Explored
  └ Read ralplan-adapter.ts, enforcement.ts, prompts.ts
    Search getDirectPlanningPrompt|getPhasePrompt\(|ralplanAdapter|getPrompt\(context in autopilot

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• getDirectPlanningPrompt() 有两条调用路径。

  1. 旧的 phase 模式

  当 autopilot state 的当前 phase 是：

  planning

  并且没有走 pipeline tracking 时，enforcement.ts 会生成当前 phase 的继续提示：

  const phasePrompt = getPhasePrompt(state.phase, ...)

  位置：oh-my-claudecode/src/hooks/autopilot/enforcement.ts:336

  然后 getPhasePrompt("planning", ...) 会调用：

  return getDirectPlanningPrompt(
    context.specPath || ".omc/autopilot/spec.md",
    context.planPath || resolveAutopilotPlanPath(),
  );

  位置：oh-my-claudecode/src/hooks/autopilot/prompts.ts:405

  所以旧模式下，调用时机是：

  autopilot Phase 0 expansion 完成
    ↓
  state.phase 转为 planning
    ↓
  Stop/persistent-mode enforcement 发现 planning 未完成
    ↓
  生成 planning prompt
    ↓
  getDirectPlanningPrompt() 被调用

  2. pipeline 模式下 planning = "direct"

  现在 autopilot 还有 pipeline-aware 路径。ralplanAdapter.getPrompt(context) 会根据配置判断：

  if (context.config.planning === "ralplan") {
    // 走 ralplan 共识规划
  } else {
    // direct planning
    ${getDirectPlanningPrompt(specPath)}
  }

  位置：oh-my-claudecode/src/hooks/autopilot/adapters/ralplan-adapter.ts:31

  pipeline enforcement 调用 adapter prompt 的地方在：

  const stagePrompt = currentAdapter.getPrompt(context);

  位置：oh-my-claudecode/src/hooks/autopilot/enforcement.ts:510

  所以 pipeline 模式下，调用时机是：

  autopilot pipeline 当前 stage = ralplan/planning adapter
    ↓
  config.planning === "direct"
    ↓
  ralplanAdapter.getPrompt(context)
    ↓
  getDirectPlanningPrompt(specPath)

  不过注意：默认 pipeline 配置是：

  planning: "ralplan"

  所以默认情况下不会走 getDirectPlanningPrompt()，而是提示调用 /oh-my-claudecode:ralplan 做 Planner /
  Architect / Critic 共识规划。

  只有当配置变成：

  {
    "autopilot": {
      "planning": "direct"
    }
  }

  或者旧 phase 模式在 planning phase 运行时，才会调用 getDirectPlanningPrompt()。

───────────────────────────────────────────────────────────────────────────────────────────────────────────────


› '/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/src/hooks/autopilot'
  这个hook 的作用，以及代码的执行顺序


• 我会按入口文件、状态管理、enforcement、pipeline/adapters 的顺序梳理，这样能把“hook 做什么”和“代码什么时候
  跑”分开讲清楚。

• Explored
  └ Read state.ts, enforcement.ts, index.ts, pipeline.ts

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 我已经读到入口和主要状态/执行模块。为了给你准确的执行顺序，我再看 persistent-mode 是在哪里调用
  checkAutopilot，以及初始化 autopilot/pipeline 的入口在哪里。

• Explored
  └ Read persistent-mode.mjs, pipeline.ts, enforcement.ts
    Search checkAutopilot\(|initAutopilot\(|initPipeline\(|autopilot-state in oh-my-claudecode

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 有个细节：templates/hooks/persistent-mode.mjs 是安装模板，src/hooks/persistent-mode/index.ts 是源码实现；我再
  看源码里的调用点，避免把模板里的简化逻辑和 TS 模块混淆。

• Explored
  └ Read pipeline-types.ts, index.ts

───────────────────────────────────────────────────────────────────────────────────────────────────────────────

• src/hooks/autopilot 不是 Claude Code 生命周期 hook 的单个脚本，而是一组 autopilot 工作流运行时模块。真正的
  Stop hook / persistent-mode hook 会调用这里的函数，用它来判断：

  autopilot 是否还没完成？
  当前在哪个阶段？
  该不该阻止 Claude 停止？
  下一步应该注入什么 prompt？
  是否要进入下一个阶段？

  模块职责

  oh-my-claudecode/src/hooks/autopilot/index.ts:1
  只是 barrel export，把 autopilot 的 state、prompt、pipeline、validation、cancel、enforcement 等函数统一导出。

  oh-my-claudecode/src/hooks/autopilot/state.ts:1
  负责 .omc/state/.../autopilot-state.json 的读写、初始化、阶段切换。

  核心函数：

  initAutopilot()
  readAutopilotState()
  writeAutopilotState()
  transitionPhase()
  transitionRalphToUltraQA()
  transitionUltraQAToValidation()
  transitionToComplete()

  oh-my-claudecode/src/hooks/autopilot/prompts.ts:1
  根据当前阶段生成给 Claude 的执行指令，比如 expansion、planning、execution、qa、validation。里面会生成包含
  Task(subagent_type=...) 的 prompt。

  oh-my-claudecode/src/hooks/autopilot/enforcement.ts:1
  这是最关键的“别停，继续跑”的逻辑。persistent-mode hook 会调用：

  checkAutopilot(sessionId, workingDir)

  它决定是否 block Stop，并返回要注入给 Claude 的 continuation prompt。

  oh-my-claudecode/src/hooks/autopilot/pipeline.ts:1
  新式 pipeline 编排器，把 autopilot 变成统一流水线：

  ralplan -> execution -> ralph -> qa

  oh-my-claudecode/src/hooks/autopilot/pipeline-types.ts:1
  定义 pipeline 配置和阶段顺序。默认配置是：

  planning: "ralplan",
  execution: "solo",
  verification: { engine: "ralph", maxIterations: 100 },
  qa: true

  oh-my-claudecode/src/hooks/autopilot/adapters/index.ts:1
  每个 adapter 把一个阶段包装成统一接口：

  ralplanAdapter
  executionAdapter
  ralphAdapter
  qaAdapter

  执行顺序

  典型顺序是这样：

  1. 用户输入 autopilot / build me / I want a
  2. keyword-detector 识别 autopilot，要求 Claude invoke autopilot skill
  3. autopilot skill 被加载
  4. 初始化 autopilot state
  5. Stop hook / persistent-mode 检查 active state
  6. checkAutopilot() 运行
  7. 根据 state 生成当前阶段 prompt
  8. Claude 执行 prompt，调用 agents / skills / commands
  9. Claude 输出阶段完成 signal
  10. 下次 Stop hook 检测 signal
  11. 进入下一阶段
  12. 重复直到 complete / failed / cancel

  旧 phase 模式顺序

  如果没有 pipeline tracking，走 legacy phase：

  expansion
    ↓ EXPANSION_COMPLETE
  planning
    ↓ PLANNING_COMPLETE
  execution
    ↓ EXECUTION_COMPLETE
  qa
    ↓ QA_COMPLETE
  validation
    ↓ VALIDATION_COMPLETE
  complete

  代码位置：

  - checkAutopilot() 读取当前 state
  - getExpectedSignalForPhase() 判断当前阶段需要哪个 signal
  - detectSignal() 从 transcript 里找 signal
  - 找到 signal 后 transitionPhase() 进入下一阶段
  - 没找到 signal 就 generateContinuationPrompt() 继续当前阶段

  新 pipeline 模式顺序

  如果 state 里有 pipeline tracking，走 pipeline-aware path：

  ralplan
    ↓ PIPELINE_RALPLAN_COMPLETE
  execution
    ↓ PIPELINE_EXECUTION_COMPLETE
  ralph
    ↓ PIPELINE_RALPH_COMPLETE
  qa
    ↓ PIPELINE_QA_COMPLETE
  complete

  关键逻辑在 checkPipelineAutopilot()：

  readPipelineTracking()
    ↓
  getCurrentStageAdapter()
    ↓
  getCurrentCompletionSignal()
    ↓
  detectPipelineSignal()
    ↓
  advanceStage() 或 currentAdapter.getPrompt()

  adapter 顺序来自：

  export const ALL_ADAPTERS = [
    ralplanAdapter,
    executionAdapter,
    ralphAdapter,
    qaAdapter,
  ]

  最关键的一点

  src/hooks/autopilot 本身不直接“执行代码修改”。它的作用是 管理状态 + 生成下一步指令 + 阻止 Claude 在未完成时停
  止。

  真正干活的是它生成的 prompt 里要求 Claude 调用的东西：

  Task(subagent_type="oh-my-claudecode:architect")
  Task(subagent_type="oh-my-claudecode:critic")
  Task(subagent_type="oh-my-claudecode:executor")
  /oh-my-claudecode:ralplan
  /oh-my-claudecode:ralph
  UltraQA 等

  所以它更像 autopilot 的状态机和续跑控制器。

─ Worked for 1m 10s ───────────────────────────────────────────────────────────────────────────────────────────


› 所以 在omc 中 hook 是自动执行的关键？


• 是的，在 OMC 里 hook 是自动执行的关键机制。

  但更准确地说：

  Skill 定义“该怎么做”
  Agent 定义“谁来做”
  State 记录“做到哪了”
  Hook 负责“什么时候自动触发、什么时候继续、什么时候阻止停止”

  尤其是这几个 hook 很关键：

  - UserPromptSubmit hook：检测 autopilot、ralph、ultrawork 等 magic keyword，然后提示 Claude 加载对应 skill。
  - Stop / persistent-mode hook：当 Claude 想结束时，检查 .omc/state/*-state.json，如果发现 autopilot/ralph 还
    没完成，就 block stop，并注入“继续当前阶段”的 prompt。
  - PreCompact hook：上下文压缩前保存状态和重要信息。
  - SubagentStart/Stop hook：跟踪 agent 执行状态。
  - PostToolUse hook：工具执行后更新状态、验证结果、记录信息。

  所以 autopilot 能“自动跑下去”，不是因为 SKILL.md 自己会执行，而是因为：

  用户触发 skill
    ↓
  skill/运行时代码创建 state
    ↓
  hook 在生命周期事件中持续检查 state
    ↓
  如果没完成，hook 阻止停止并注入下一步指令
    ↓
  Claude 继续执行

  一句话：Skill 是策略，hook 是自动化触发和续跑引擎。