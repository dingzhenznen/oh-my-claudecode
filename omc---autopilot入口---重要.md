入口有三层，容易混在一起：

  bridge  main

  1. 总入口：processHook()

  所有 hook 都先进这里：

  oh-my-claudecode/src/hooks/bridge.ts:2531

  export async function processHook(hookType, rawInput) {
    ...
    switch (hookType) {
      case "keyword-detector":
        return await processKeywordDetector(input);

      case "persistent-mode":
        return await processPersistentMode(input);

      case "autopilot":
        return await processAutopilot(input);
    }
  }

  也就是说，Claude Code 触发某个生命周期 hook 后，OMC bridge 根据 hook type 分发。

  2. 启动入口：keyword-detector

  用户输入：

  autopilot build me ...

  先走 keyword-detector，它检测到 autopilot keyword：

  oh-my-claudecode/templates/hooks/keyword-detector.mjs:772

  if (hasActionableKeyword(cleanPrompt, /\b(autopilot|auto[\s-]?pilot|fullsend|full\s+auto)\b/i)) {
    matches.push({ name: 'autopilot', args: '' });
  }

  然后它向 Claude 注入类似：

  [MAGIC KEYWORD: AUTOPILOT]
  Skill: oh-my-claudecode:autopilot

  同时测试显示它会 seed 一个 inert autopilot state：

  .omc/state/sessions/{sessionId}/autopilot-state.json

  但这个 state 初始有：

  awaiting_confirmation: true

  所以真正 skill 没确认前，Stop hook 不会误阻塞。

  3. 续跑入口：persistent-mode → checkAutopilot()

  这是 autopilot 自动继续的主入口。

  persistent-mode 在 Stop 事件时运行，调用：

  oh-my-claudecode/src/hooks/persistent-mode/index.ts:1859

  const autopilotResult = await checkAutopilot(sessionId, workingDir);

  checkAutopilot() 定义在：

  oh-my-claudecode/src/hooks/autopilot/enforcement.ts:207

  它才是 src/hooks/autopilot 这套状态机的核心入口：

  readAutopilotState()
    ↓
  如果 inactive / session 不匹配 / awaiting_confirmation，返回 null
    ↓
  如果有 pipeline tracking，走 checkPipelineAutopilot()
    ↓
  否则走 legacy phase enforcement
    ↓
  检测 signal
    ↓
  切换阶段或生成 continuation prompt

  4. 独立 autopilot hook 入口：processAutopilot()

  还有一个 hook type 叫 "autopilot"，入口是：

  oh-my-claudecode/src/hooks/bridge.ts:2470

  async function processAutopilot(input) {
    const { readAutopilotState, getPhasePrompt } =
      await import("./autopilot/index.js");

    const state = readAutopilotState(directory, input.sessionId);

    if (!state || !state.active) {
      return { continue: true };
    }

    const phasePrompt = getPhasePrompt(state.phase, context);

    return {
      continue: true,
      message: `[AUTOPILOT - Phase: ${state.phase.toUpperCase()}]\n\n${phasePrompt}`,
    };
  }

  这个入口的作用比较轻：如果已经有 active autopilot state，就按当前 phase 注入 prompt。它不像 persistent-mode
  那样负责强制阻止停止。

  所以调用顺序可以理解成

  用户输入
    ↓
  Claude Code UserPromptSubmit
    ↓
  OMC bridge.processHook("keyword-detector")
    ↓
  processKeywordDetector()
    ↓
  检测 autopilot，要求加载 Skill("oh-my-claudecode:autopilot")
    ↓
  autopilot skill / state 初始化
    ↓
  Claude 执行一轮
    ↓
  Claude 想停止
    ↓
  Claude Code Stop
    ↓
  OMC bridge.processHook("persistent-mode")
    ↓
  processPersistentMode()
    ↓
  checkAutopilot()
    ↓
  生成下一阶段/当前阶段 continuation prompt

  一句话：src/hooks/autopilot 最重要的运行入口是 checkAutopilot()，它由 persistent-mode Stop hook 调用；
  processAutopilot() 是 bridge 里的独立 autopilot hook 分支；keyword-detector 是最开始发现并激活 autopilot 的入
  口。