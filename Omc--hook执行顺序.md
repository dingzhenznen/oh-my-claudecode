'/Users/dingzhenzhen/works/study/dingzhengit/oh-my-claudecode/src/hooks/autopilot'
  这个hook 的作用，以及代码的执行顺序
  
我会按入口文件、状态管理、enforcement、pipeline/adapters 的顺序梳理，这样能把“hook 做什么”和“代码什么时候
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