OMC 在 Claude Code 中的运行机制总结                                  
                                             
  五层架构                                                                
  
  用户输入                                                                
      ↓                                      
  【Hook 层】检测关键词 (keyword-detector.mjs)
      ↓                                                                   
  【Skill 层】注入行为增强 (ultrawork / ralph 等)
      ↓                                                                   
  【系统提示词层】注入 omcSystemPrompt + 19 个代理
      ↓                                                                   
  【Agent 执行层】并行协调 19 个专精代理     
      ↓                                                                   
  【工具和状态层】MCP 服务、权限、任务追踪   
                                                                          
  ---                                        
  核心运作流程                                                            
                                             
  当用户输入 "ultrawork refactor the authentication module" 时：

  1. Hook 捕获 (立即)
    - keyword-detector.mjs 检测到 "ultrawork" 关键词
    - 返回增强指令
  2. 系统提示词构建 (会话初始化)
  // src/index.ts - createOmcSession()
  let systemPrompt = omcSystemPrompt  // "You are the relentless 
  orchestrator..."                                                        
  systemPrompt += continuationAddition  // 防止中途停止
  systemPrompt += ultraworkEnhancement  // ultrawork 并行模式             
  systemPrompt += projectContext  // AGENTS.md + CLAUDE.md
  3. 19 个代理就绪
    - executor (sonnet): 实现主逻辑
    - git-master (sonnet): 原子提交
    - verifier (sonnet): 完成验证                                         
    - 其他: 按需激活
  4. 并行执行                                                             
    - BackgroundTaskManager 管理并发         
    - 允许多个代理同时工作
    - executor 实现代码 + git-master 准备提交
  5. 验证完成
    - verifier 检查: [✓] 代码修改 [✓] 测试通过                            
    - 如果验证失败 → 重新执行
    - 系统提示词中的检查清单强制完成                                      
                                             
  ---
  关键创新点
            
  ┌────────────────┬──────────────────────────────┬───────────────────┐
  │      特性      │             机制             │       好处        │   
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 并行执行       │ BackgroundTaskManager +      │ 快速完成复杂任务  │   
  │                │ 多代理                       │                   │
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 续航执行       │ 系统提示词中的检查清单       │ 防止任务中途停止  │
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 魔法关键词     │ detectMagicKeywords +        │ 自动激活特定技能  │   
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 魔法关键词     │ detectMagicKeywords +        │ 自动激活特定技能  │
  │                │ processPrompt                │                   │
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 模型路由       │ haiku/sonnet/opus 动态选择   │ 成本优化 +        │
  │                │                              │ 质量保证          │
  ├────────────────┼──────────────────────────────┼───────────────────┤
  │ 上下文自动注入 │ loadContextFromFiles +       │ 项目信息自动融合  │
  │                │ AGENTS.md                    │                   │
  └────────────────┴──────────────────────────────┴───────────────────┘

  ---
  三个关键文件

  1. src/index.ts - 会话工厂
    - createOmcSession(): 返回完整配置对象给 Claude Agent SDK
  2. src/agents/definitions.ts - 代理工厂
    - omcSystemPrompt: 系统提示词核心
    - getAgentDefinitions(): 19 个代理定义
  3. src/installer/index.ts - 安装程序
    - install(): 将 Hook 脚本、代理、技能等安装到 ~/.claude/

  ---
  核心概念

  OMC = Hook系统 + Skill系统 + Agent系统 + State系统
        (事件检测)  (行为注入)   (执行协调)   (进度追踪)

  我已经生成了详细的架构分析文档并保存到记忆系统，包含了完整的流程图、代码
  示例和所有关键组件的详细说明