'/Users/dingzhenzhen/works/xuehua/newApi-ding/oh-my-claudecode/skills/autopilot/SKILL.md'\                                  帮我分析这个 skill                                                                                                          
  ⎿  ⧉ Selected 1 lines from ../sss.tx in Visual Studio Code                                                                
                                                                                                                            
  Read 1 file (ctrl+o to expand)                                                                                          
                                                                                                                            
⏺ autopilot SKILL 分析                                                                                                      
                                                                                                                            
  定位                                                                                                                      
                                                                                                                            
  这是 OMC 最高级别（level: 4）的 skill，从一句话描述到可运行代码的全自动流水线。                                           
   
  ---                                                                                                                       
  执行流程（6 个阶段）                                                                                                    
                                                                                                                            
  Phase 0: Expansion    → 需求分析 + 技术规格 (.omc/autopilot/spec.md)
  Phase 1: Planning     → 实现计划 (.omc/plans/autopilot-impl.md)                                                           
  Phase 2: Execution    → Ralph + Ultrawork 并行实现                                                                        
  Phase 3: QA           → 构建/测试/修复循环（最多 5 轮）
  Phase 4: Validation   → Architect + Security + Code 三方并行审查
  Phase 5: Cleanup      → 清理状态文件

  ---
  3 个智能跳过机制

  ┌──────────────────────────┬──────────────────────────────────┐
  │           条件           │               跳过               │
  ├──────────────────────────┼──────────────────────────────────┤
  │ 已有 ralplan 共识计划    │ 跳过 Phase 0 + Phase 1，直接执行 │
  ├──────────────────────────┼──────────────────────────────────┤
  │ 已有 deep-interview spec │ 跳过 Phase 0 分析，直接规划      │
  ├──────────────────────────┼──────────────────────────────────┤
  │ 输入模糊                 │ 主动询问是否先跑 /deep-interview │
  └──────────────────────────┴──────────────────────────────────┘

  最佳实践链路

  /deep-interview "模糊想法"
      → 苏格拉底式澄清 → spec（歧义 ≤ 20%）
      → /ralplan → 三方验证计划
      → /autopilot → 直接从 Phase 2 开始执行

  ---
  关键设计决策

  1. 失败止损：同一个 QA 错误出现 3 次 → 停止并报告，不无限循环
  2. 验证门控：Phase 4 三个审查者必须全部通过，否则修复后重审
  3. 可恢复：中途取消后重新运行 /autopilot 会从上次中断处继续
  4. 可配置：.claude/omc.jsonc 可以关闭 QA/Validation 阶段，调整迭代上限

  ---
  适用 vs 不适用

  ┌────────────────────┬────────────────┐
  │    用 autopilot    │ 不用 autopilot │
  ├────────────────────┼────────────────┤
  │ 新功能从零开始     │ 修一个小 bug   │
  ├────────────────────┼────────────────┤
  │ 多文件、多阶段任务 │ 单个文件修改   │
  ├────────────────────┼────────────────┤
  │ 需要测试+验证      │ 只是探索方案   │
  ├────────────────────┼────────────────┤
  │ 描述清晰的产品需求 │ 开放性问题     │
  └────────────────────┴────────────────┘

  ---
  总结

  autopilot 本质是一个元编排器，它自己不写代码，而是按顺序调度 Analyst/Architect/Executor/Security-reviewer/Code-reviewer
  等专门 agent 各司其职，最后交付经过测试和多方审查的代码。