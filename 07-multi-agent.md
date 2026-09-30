---
title: 第 8 章：多 Agent 架构（朗读版）
---

第 8 章：多 Agent 架构

本文是 docs/07-multi-agent.md 的朗读版，表格与代码已转为口语描述，内容未增删。

从单个 Agent 到 Agent 团队——Claude Code 如何协调多个 Agent 并行完成复杂任务。

8.1 三种多 Agent 模式

Claude Code 支持三种多 Agent 协作模式，适用于不同复杂度的场景。示意图展示了这三种模式的结构。首先是模式一，子 Agent 模式：父 Agent 通过 fork 派生子 Agent，子 Agent 完成后把结果返回给父 Agent。然后是模式二，协调器模式：协调器只分配不执行，向三个 Worker 派生任务，各 Worker 再把结果交回协调器。最后是模式三，Swarm 团队：多个 Agent 两两之间通过信箱对等通信。

三种模式的对比：第一，子 Agent 模式，适用于单个独立子任务，通信方式是 fork 加返回，特点是最简单，父 Agent 等待结果。第二，协调器模式，适用于复杂多步任务，通信方式是派生加综合，特点是协调器不执行，只编排。第三，Swarm 团队模式，适用于并行协作任务，通信方式是命名信箱，特点是 Agent 间对等通信。

这三种模式的复杂度递增，但共享同一套底层基础设施——AgentTool 工具、ToolUseContext 上下文隔离和 task-notification 结果通知。理解子 Agent 模式是理解后两种模式的基础。

选择多 Agent 模式的决策指南：如果只是简单的独立子任务，选子 Agent 模式，这是最简单的选择。如果需要子任务的输出作为后续输入，也选子 Agent 模式，它是同步的，由父 Agent 串行编排。如果需要多个 Worker 并行处理不同任务，还要再细分：需要中央编排、综合结果的，选 Coordinator 模式；Agent 之间对等协作、无中心的，选 Swarm 模式。如果需要执行前审批计划，选 Plan 模式，它可以与上述任何模式组合。如果不确定，就从子 Agent 模式开始，复杂度不够时再升级。

8.2 子 Agent 模式（AgentTool）

这是最基础的多 Agent 模式。父 Agent 通过 AgentTool 派生子 Agent 执行独立任务，AgentTool 的完整设计详见第 4 章。

关键文件：src/tools/AgentTool/AgentTool.tsx

完整参数解析：（代码从略：这段代码列出了 AgentTool 的全部参数。要点是 description 为三到五个词的任务描述，prompt 为完整任务指令，两者必填，且 Worker 从零开始、没有对话上下文；subagent_type 指定专用 Agent 类型；model 可指定 sonnet、opus 或 haiku 来覆盖默认模型；run_in_background 控制异步执行，结果通过 task-notification 通知；name 是用于 SendMessage 的可寻址名称；isolation 可选 worktree 或 remote 隔离模式。）

关键设计：prompt 必须是自包含的——Worker 无法看到父 Agent 的对话历史。这意味着每个 prompt 都需要包含完成任务所需的全部信息：文件路径、行号、具体的修改内容。

为什么采用这种"无上下文"设计而非共享对话历史？原因有三。第一是隔离性，子 Agent 不会被父 Agent 对话中无关的信息干扰，上下文更加聚焦。第二是成本控制，共享完整对话历史会大幅增加每次 API 调用的 token 消耗。第三是并行安全，多个子 Agent 并行运行时，如果共享可变的对话历史会引发竞态条件。

唯一的例外是 Fork 子 Agent（后文详述），它通过精巧的缓存机制在继承完整上下文的同时保持了经济性。

子 Agent 类型系统

subagent_type 决定了 Worker 的工具集、系统提示词和行为约束。Claude Code 源码中定义了三层 Agent 类型。

第一层：内建类型，位于 src/tools/AgentTool/built-in/ 目录。这些类型由 Claude Code 核心代码定义，经过精心优化。三种内建类型分别是：第一种 general-purpose，工具集为全部工具，使用默认子 Agent 模型，系统提示词最简化，只要求完成任务、简洁汇报，用于通用任务。第二种 Explore，工具集排除 Agent、Edit、Write、NotebookEdit，外部用户使用 Haiku 模型以求快，内部用户继承父级模型，系统提示词是严格只读加并行搜索优化，用于代码库探索。第三种 Plan，工具集与 Explore 相同，继承父级模型，系统提示词是只读加结构化输出要求，用于设计实施方案。

第二层：自定义类型，来自 .claude/agents/ 目录下的 Markdown 文件。用户通过 Markdown frontmatter 定义，支持所有 BaseAgentDefinition 字段。（代码从略：这段示例是一个自定义 Agent 的定义文件。要点是通过 frontmatter 声明它是数据库迁移专家，只允许 Bash、Read、Edit 三个工具，模型指定为 sonnet，权限模式为 plan，正文再写明它的角色设定。）

第三层：插件类型。通过插件系统注入，具有 source 为 plugin 的标识。

Explore Agent 深度分析

Explore Agent 的设计体现了多个精细的工程取舍，源码位于 src/tools/AgentTool/built-in/exploreAgent.ts。

系统提示词的 READ-ONLY 硬约束：提示词开头就用 CRITICAL: READ-ONLY MODE 这样的显式声明列出禁止列表——不能创建、修改、删除文件，不能用重定向写文件，不能运行改变系统状态的命令。虽然 disallowedTools 已经在工具层面阻止了写入工具，但系统提示词的重复声明是为了在模型层面增加一道安全屏障——模型不会尝试通过 Bash 工具间接写文件。

Haiku 模型选择：外部用户使用 Haiku，速度优先；内部用户继承父级模型。这个选择基于 Explore 的任务特性——搜索和读取文件不需要强推理能力，速度更重要。源码中的注释解释了这一点：内部用户的模型设为 inherit，即继承父级；外部用户则用 haiku 换取速度。

omitClaudeMd 设为 true 的成本优化：Explore Agent 不需要知道项目的 commit 规范、PR 模板等 CLAUDE.md 中的规则——它只读代码，由父 Agent 解读结果。源码注释揭示了这个优化的规模。（代码从略：这段代码把 omitClaudeMd 设为 true，注释解释说 Explore 是一个快速的只读搜索 Agent，不需要 CLAUDE.md 里的 commit、PR 和 lint 规则，主 Agent 拥有完整上下文并负责解读结果。）

在每周 34M 次以上的 Explore 调用规模下，省略 CLAUDE.md 每周可节省约 5 到 15 Gtok。

并行工具调用的速度提示：系统提示词末尾特别强调"尽可能并行调用多个工具进行搜索和文件读取"——这是利用 API 的并行工具调用能力来加速搜索。

Plan Agent 深度分析

Plan Agent（src/tools/AgentTool/built-in/planAgent.ts）与 Explore 共享只读工具限制，但有不同的设计目标。

结构化输出要求：系统提示词要求 Plan Agent 在输出末尾必须包含 Critical Files for Implementation 列表，也就是三到五个文件。这不是可选建议——它确保规划结果是可操作的，父 Agent 能根据这些关键文件路径开始执行。

继承父级模型：与 Explore 使用 Haiku 不同，Plan 的 model 设为 inherit，因为架构设计和方案规划需要更强的推理能力。

工具列表复用：tools 直接引用 EXPLORE_AGENT.tools——Plan 直接引用 Explore 的工具定义，确保两者保持一致。

General-purpose Agent 设计哲学

General-purpose Agent（src/tools/AgentTool/built-in/generalPurposeAgent.ts）的设计哲学是"最小约束"。（代码从略：这段代码定义了共享的提示词前缀。要点是要求 Agent 完整完成任务——不要过度打磨，但也不要半途而废。）

它的具体表现有四条：tools 配置为全部工具，赋予全部工具能力；不设置 omitClaudeMd，因为通用 Agent 可能需要遵守项目的 commit 规范等规则；不指定 model，由 getDefaultSubagentModel 函数获取默认子 Agent 模型；系统提示词简洁，只要求"完成任务，简洁汇报"。

为什么限制工具集？不同任务有不同的安全需求。Explore Agent 只需要读取代码，赋予它写入能力是不必要的风险。类型系统实现了最小权限原则。

AgentTool 调用完整流程

当模型发出一次 Agent 工具调用时，系统经历以下 5 个阶段。理解这个流程有助于理解为什么子 Agent 能做到既隔离又高效。流程图描述：模型发出 Agent 工具调用，参数包含 description、prompt 和 subagent_type。首先进行类型解析，查找对应的 AgentDefinition。然后依次组装工具池，先执行 assembleToolPool 再做 filterToolsForAgent 过滤；构建系统提示词，先取 getSystemPrompt 再用环境细节增强；随后创建上下文。最后进入执行分支：同步时直接执行，阻塞父级等待结果；异步时注册异步 Agent，立即返回 taskId；Worktree 分支创建隔离文件系统；远程分支则传送到 CCR 环境。

阶段 1：类型解析

类型解析的核心逻辑见 AgentTool.tsx 中的 effectiveType 决策段落。（代码从略：这段代码处理 fork 子 Agent 实验的路由。要点是显式指定了 subagent_type 就直接使用它；省略类型且 fork 实验开启时走 fork 路径；省略类型且实验关闭时回退到 general-purpose。）

这段代码按三种情况分派：显式指定类型，直接使用，不猜测——即 explicit wins；省略类型且 fork 实验开启，走 fork 路径，继承完整上下文；省略类型且 fork 实验关闭，回退到 general-purpose。

如果指定了类型，系统从 agentDefinitions.activeAgents 列表中查找匹配的 AgentDefinition。找不到时，会区分"不存在"和"被权限拒绝"两种情况，给出不同的错误提示——这对用户调试很有帮助。

阶段 2：工具池组装

子 Agent 的工具池独立于父级构建，这是一个关键的隔离设计（见 AgentTool.tsx 的 568 到 577 行）。（代码从略：这段代码独立组装 Worker 的工具池。要点是 Worker 的工具总是用自己的权限模式经 assembleToolPool 生成，不受父级工具限制的影响；其权限模式取自所选 Agent 定义的 permissionMode，缺省为 acceptEdits。）

注意 permissionMode 默认是 acceptEdits——这意味着子 Agent 默认情况下可以自动执行编辑操作，无需逐个确认。这是合理的，因为子 Agent 已经由父 Agent 委托了明确的任务。

工具池组装后，还要经过 filterToolsForAgent 的多层过滤（详见下文"工具过滤流水线"）。

阶段 3：系统提示词构建

普通子 Agent 和 Fork 子 Agent 的提示词构建路径完全不同（见 AgentTool.tsx 的 483 到 541 行）。

普通路径分三步：第一步，调用 agent 定义的 getSystemPrompt 函数获取基础提示词；第二步，用 enhanceSystemPromptWithEnvDetails 追加环境信息，包括绝对路径格式、平台信息等；第三步，用户的 prompt 作为一条独立的 user 消息发送。

Fork 路径分两步：第一步，直接使用父级已渲染的系统提示词字节，即 toolUseContext.renderedSystemPrompt，不重新计算；第二步，用 buildForkedMessages 构建消息序列，克隆父级 assistant 消息，加上占位 tool_result 和子级指令。

Fork 路径为什么不重新计算系统提示词？因为 GrowthBook，也就是 A/B 测试系统，的状态可能在父级 turn 开始和 fork 生成之间发生变化，重新计算会产生不同的字节序列，导致 Prompt Cache 失效。

阶段 4：上下文创建

createSubagentContext 函数（src/utils/forkedAgent.ts 的 345 到 462 行）是整个多 Agent 架构的安全基石。详见下文"上下文隔离深度解析"。

阶段 5：执行分支

执行模式的选择逻辑在 AgentTool.tsx 的 555 到 567 行。（代码从略：这段代码判断是否应该异步运行。要点是满足任一条件即异步——调用方传了 run_in_background；所选 Agent 定义本身是后台类型；协调器模式下所有 Agent 都异步；fork 实验开启时所有 Agent 都异步；助手模式下强制异步。同时要求后台任务功能未被禁用。）

几个值得注意的设计：协调器模式强制异步，因为协调器需要同时管理多个 Worker，同步执行会阻塞编排；Fork 实验强制异步，统一使用 task-notification 交互模型；进程内队友不能运行后台 Agent，因为其生命周期绑定到父级，强制后台会导致孤儿进程。

工具过滤流水线

子 Agent 的工具要经过一条精心设计的四层过滤流水线，而不是简单地"给什么用什么"。这条流水线实现了纵深防御：即使某一层有漏洞，其他层仍能拦截危险工具访问。

关键函数：filterToolsForAgent，位于 src/tools/AgentTool/agentToolUtils.ts 的 70 到 116 行。

流水线流程描述：所有可用工具首先进入第一层，由 ALL_AGENT_DISALLOWED_TOOLS 移除 TaskOutput、EnterPlanMode、AskUserQuestion 等元工具，这些是"元工具"，只有父级应该使用。然后判断是否为内建 Agent，不是的话进入第二层，由 CUSTOM_AGENT_DISALLOWED_TOOLS 对非内建 Agent 额外限制。接着判断是否为异步 Agent，是的话进入第三层白名单 ASYNC_AGENT_ALLOWED_TOOLS，只允许 Read、Grep、Glob、Edit、Write、Bash、Skill 等工具。最后是第四层，应用 Agent 自身的 disallowedTools，例如 Explore 排除 FileEdit 和 FileWrite，得到最终工具集。另有两条旁路：MCP 工具始终放行，ExitPlanMode 在 plan 模式下放行。

第一层 ALL_AGENT_DISALLOWED_TOOLS：移除"元工具"——TaskOutput、EnterPlanMode、ExitPlanMode、AskUserQuestion、TaskStop 等。这些工具用于控制 Agent 的执行流程本身，子 Agent 不应该能进入 Plan 模式或向用户提问。

第二层 CUSTOM_AGENT_DISALLOWED_TOOLS：对用户自定义的 Agent（来自 .claude/agents/ 目录）施加额外限制。这是一个安全边界——用户定义的 Agent 类型不应该获得与内建类型相同的权限。

第三层 ASYNC_AGENT_ALLOWED_TOOLS 是白名单模式：异步 Agent 只能用白名单里的 Read、Grep、Glob、Edit、Write、Bash、Skill、NotebookEdit 等工具。为什么异步 Agent 要受更严的限制？因为它在后台运行，弹不出交互式 UI，像权限确认弹窗就显示不出来，凡是需要用户交互的工具都得排除。

第三层的例外有三条。其一，MCP 工具（名称以 mcp__ 开头）始终放行——它们由用户配置的外部服务提供，用户对其安全性负责。其二，ExitPlanMode 在 permissionMode 为 plan 时允许——进程内队友需要退出 Plan 模式的能力。其三，进程内队友额外拿到 Agent 工具和一组任务协调工具，包括 TaskCreate、TaskGet、TaskList、TaskUpdate 和 SendMessage。有了 Agent 工具，它能派生同步子 Agent；有了协调工具，它能共享任务列表、和别的队友互通消息。任务系统的完整分析详见第 11 章。

第四层：Agent 自身定义的 disallowedTools。例如 Explore Agent 显式排除 Agent、ExitPlanMode、FileEdit、FileWrite、NotebookEdit 这些工具。

设计洞察：前三层是全局策略，约束所有 Agent；第四层是类型级策略，只约束特定类型。这样分层，即使有人写了一个 disallowedTools 为空列表的自定义 Agent，它仍然受前三层保护。

上下文隔离深度解析

createSubagentContext 函数（src/utils/forkedAgent.ts 的 345 到 462 行）是多 Agent 架构的安全基石。它为每个子 Agent 创建一个隔离的 ToolUseContext，确保子 Agent 的行为不会影响父级。

核心设计原则是"默认隔离，显式共享"，也就是 deny by default：所有可变状态默认是隔离的，如果需要共享必须通过 shareSetAppState、shareAbortController 等参数显式开启。

示意图的朗读描述：父级和子级各有一个 ToolUseContext，图中对比了七个字段的处理方式。readFileState 通过 cloneFileStateCache 克隆后传给子级；abortController 通过 createChildAbortController 新建子控制器；getAppState 经过包装；setAppState 被替换为空操作；setAppStateForTasks 直接共享；queryTracking 换用新的 UUID 并把深度加一；contentReplacementState 通过克隆函数传给子级。

逐项解析每个字段的隔离方式和设计原因：

readFileState：克隆。源码中，readFileState 取父级的缓存或调用方覆盖值，再经 cloneFileStateCache 克隆一份。文件状态缓存记录了每个文件的最后读取时间和内容哈希。如果子 Agent 与父级共享同一个缓存，子 Agent 的文件读取会改变缓存状态，导致父级对文件新鲜度的判断出错。克隆确保子 Agent 的读取操作不会"污染"父级的缓存。

abortController：新建子控制器。（代码从略：这段代码决定 abortController 的来源。要点是调用方可以显式传入；如果要求共享 abort，就直接复用父级的控制器；否则用 createChildAbortController 新建一个链接到父级的子控制器。）

createChildAbortController 使用 WeakRef 创建一个链接到父级的子控制器。关键行为有两条：父级中断，子级也中断，通过事件监听器传播 abort 信号；反过来，子级中断不等于父级中断，子级的 abort 只清理自己的监听器，不影响父级。这个单向传播是故障隔离的基础：一个子 Agent 的失败（被 abort）不会连锁影响父级或其他子 Agent。

getAppState：包装。（代码从略：这段代码决定 getAppState 的来源。要点是交互式子 Agent 直接共享父级的 getAppState；非交互式子 Agent 则拿到一个包装版本，它在父级状态的基础上强制把 shouldAvoidPermissionPrompts 设为 true。）

非交互式子 Agent（后台运行）的 getAppState 被包装为始终返回 shouldAvoidPermissionPrompts 为 true。这防止后台子 Agent 弹出权限确认对话框阻塞父级的终端——后台 Agent 没有地方显示 UI。

setAppState：默认 no-op。（代码从略：这段代码决定 setAppState 的来源。要点是显式要求共享时才复用父级的 setAppState，否则替换为空函数，子 Agent 的状态变更不传播。）

子 Agent 的状态变更（如工具进度、响应长度）默认不会传播到父级 UI。这避免了多个并行子 Agent 同时更新 UI 导致的混乱。

setAppStateForTasks：始终共享。（代码从略：这段代码把 setAppStateForTasks 指向父级的对应回调。注释强调，任务的注册与终止必须始终到达根 store，即使 setAppState 是空操作——否则异步 Agent 的后台 bash 任务永远不会被注册，也永远不会被终止。）

这是唯一一个即使 setAppState 是空操作也必须共享的回调。为什么？因为子 Agent 可能通过 Bash 工具启动后台进程。如果这些进程的注册信息到不了根 store，当子 Agent 结束时这些进程就成了僵尸进程——PPID 为 1，无人回收。

queryTracking：新 chainId 加深度加一。（代码从略：这段代码为每个子 Agent 生成一个新的随机链路 ID 作为 chainId，并把父级深度加一作为自己的深度。）

这个字段有两个作用：第一，防止无限递归，depth 递增使系统能够检测和限制 Agent 嵌套深度；第二，链路追踪，chainId 允许分析系统追踪 Agent 的家族谱系，用于性能分析和调试。

contentReplacementState：克隆而非新建。（代码从略：这段代码默认克隆父级的 contentReplacementState 而不是新建。注释解释说，缓存共享的 fork 会处理包含父级 tool_use_id 的消息，全新的状态会把它们当成没见过，做出不同的替换决策，导致请求前缀不同、缓存失效。）

这个字段的处理方式特别精妙。它管理工具结果中的内容替换（如截断超长输出）。为什么用克隆而不是新建？因为 Fork 子 Agent 会处理包含父级 tool_use_id 的消息。如果用一个全新的状态，对同一个 tool_use_id 会做出不同的替换决策，导致 API 请求的字节序列不同——Prompt Cache 就失效了。克隆确保对已知 ID 做出相同的决策，维持缓存命中。

四种执行模式

四种执行模式的对比：第一，同步模式，进程内直接执行，结果嵌入父对话，适用于简单子任务。第二，异步模式，基于 LocalAgentTask 实现，结果以 task-notification XML 传递，适用于长时间任务。第三，队友模式，通过 Tmux、iTerm2 或 InProcess 会话实现，以信箱通信，适用于并行协作。第四，远程模式，基于 RemoteAgentTask 实现，通过 WebSocket 流式传输，适用于 CCR 环境。

同步模式是最简单的：父 Agent 阻塞等待子 Agent 完成，结果直接作为 tool_result 嵌入父级对话。适合快速的探索或搜索任务。

异步模式适合长时间运行的任务。registerAsyncAgent 在 AppState.tasks 中注册任务状态，父 Agent 立即收到一个包含 agentId 和 outputFile 的响应，可以继续处理其他工作。任务完成时，enqueueAgentNotification 将 task-notification XML 作为 user 角色消息投递到父级的下一轮对话中。

自动后台化：当同步 Agent 运行超过 120 秒（由 getAutoBackgroundMs 函数决定），系统自动将其转为后台任务，避免长时间阻塞父级。（代码从略：这段代码定义自动后台化的毫秒数。要点是当环境变量 CLAUDE_AUTO_BACKGROUND_TASKS 为真，或 tengu_auto_background_agents 特性开关开启时，返回 120000 毫秒，即 120 秒；否则返回 0。）

隔离模式

Git Worktree 隔离：子 Agent 在独立的 Git Worktree 中工作，防止多个 Agent 同时修改同一文件。目录结构是这样的：主仓库在 main 分支上，Agent A 直接在这里工作；同时 .git 的 worktrees 目录下有 worktree-abc 和 worktree-def 两个子目录，分别是 Agent B 和 Agent C 的隔离副本。

Worktree 创建过程（src/utils/worktree.ts）分三步。第一步，Slug 验证：最长 64 字符，只允许字母数字和点、斜杠、连字符、下划线，禁止路径穿越，包括两点相对路径和绝对路径——这是安全边界，防止子 Agent 通过 slug 注入访问仓库外的文件。第二步，创建：在 .claude/worktrees/ 目录下按 slug 创建，对 node_modules 这类大目录使用符号链接避免磁盘占用。第三步，清理：任务完成后，如果 worktree 无任何文件变更（通过 git diff 检测），自动删除；有变更时返回路径和分支名，由用户决定是否合并。

远程隔离：在远程 CCR 环境中执行，CCR 即 Claude Code Remote，Anthropic 的云端远程会话。它通过 WebSocket 流式传输消息，适用于需要完全隔离的沙盒环境。远程隔离始终以异步模式运行。

Fork 子 Agent

当 subagent_type 未指定且 FORK_SUBAGENT feature gate 启用时，系统创建 fork 子 Agent——一种特殊模式，继承父级完整对话上下文。流程描述：首先，父 Agent 的对话上下文被字节精确复制给 Fork 子 Agent，这有利于缓存复用；然后，Fork 子 Agent 继承相同的系统提示词和完整的消息历史；最后，它独立执行，并把结果返回父级。

为什么需要 Fork？Prompt Cache 的经济学

Fork 机制的核心动机是 Prompt Cache 共享。理解这一点需要先理解 Anthropic API 的缓存机制。

API 按请求前缀缓存，前缀包括系统提示词、工具定义和消息前缀。如果两个请求的前缀字节完全相同，第二个请求可以复用第一个的缓存，缓存读取的 token 比 input token 便宜 90%。

普通子 Agent 自带一份系统提示词，消息历史是空的，跟父级的请求前缀完全不同，没法共享缓存。每次调用都是"冷启动"。

Fork 子 Agent 则不同：它继承父级的完整请求前缀——相同的系统提示词、相同的工具定义、相同的消息历史，只在末尾追加一条不同的指令。这意味着所有从同一个父级 fork 出来的子 Agent 都共享同一个缓存前缀——第一个 fork 是冷启动，后续的都是缓存命中。

源码中的 CacheSafeParams 类型（forkedAgent.ts 的 57 到 68 行）明确了这个"字节级相同"的要求。（代码从略：这个类型定义了 fork 所需的全部参数。注释逐一强调：systemPrompt 必须与父级一致才能命中缓存；userContext 会前置到消息里，影响缓存；systemContext 会追加到系统提示词后，同样影响缓存；toolUseContext 包含工具、模型等选项；forkContextMessages 是用于共享 prompt 缓存的父级上下文消息。）

Fork 消息构建

buildForkedMessages 函数（forkSubagent.ts 的 107 到 169 行）是 fork 机制的核心——它构建一组消息，确保所有 fork 子级的请求前缀字节相同。

消息结构描述：所有 fork 共享一个前缀，也就是缓存命中区。它由三部分组成：先是历史消息；然后是父级的 assistant 消息，包含所有 tool_use、thinking 和 text 内容块；再是一条 user 消息，为每个 tool_use 放一个占位 tool_result，文本统一为 Fork started——正在后台处理。在这条共享前缀之后，每个 fork 只追加一段自己独有的指令，比如 Fork A 是搜索所有 TODO 注释，Fork B 是分析测试覆盖率。

关键实现细节有三条。第一，克隆父级 assistant 消息：保留所有内容块——thinking、text 和每个 tool_use，不做修改，确保字节相同。第二，占位 tool_result：为每个 tool_use 生成一个 tool_result，文本统一为 Fork started——processing in background，即"fork 已启动，正在后台处理"。为什么不用实际结果？因为实际结果各不相同，会破坏缓存前缀的一致性。第三，每个子级独有的 directive：只有最后一个文本块是每个 fork 独有的，包含该 fork 需要执行的具体指令。

递归 Fork 防护

Fork 子级的工具池中保留了 Agent 工具——为了缓存一致性，如果移除会改变工具定义的字节——但在运行时通过两道防线阻止递归 fork。（代码从略：这段代码是两道检测条件的或运算。第一道检查 querySource 是否为内建 fork Agent 的来源标记，第二道调用 isInForkChild 扫描消息历史中是否含有 FORK_BOILERPLATE_TAG 标记。）

为什么需要两道？querySource 是在 context 的 options 中设置的，不受消息自动压缩的影响——这是首选方案。消息扫描是后备方案，覆盖 querySource 没有被正确传递的边缘情况。

Fork Agent 定义

（代码从略：这段代码定义了 FORK_AGENT 常量。要点是 agentType 为 fork；tools 为全部工具，保持与父级缓存一致；maxTurns 上限 200 轮；model 为 inherit，继承父级模型，保证上下文长度对等；permissionMode 为 bubble，权限请求冒泡到父级终端；getSystemPrompt 返回空字符串，因为它不会被使用——fork 直接使用父级已渲染的系统提示词。）

permissionMode 为 bubble 是一个独特的权限模式——当 fork 子级需要权限确认时，请求会"冒泡"到父级的终端显示，而不是被静默拒绝。这是因为 fork 子级被设计为"父级的延伸"，它的操作在概念上仍然由用户控制。

getSystemPrompt 返回空字符串看起来像一个 bug，但实际上是刻意设计——fork 路径从不调用这个函数，而是直接传入父级的 renderedSystemPrompt 字节。如果不小心调用了它（比如代码路径错误），空字符串会导致明显的异常，而不是一个微妙的缓存失效。

与协调器模式互斥：Fork 和协调器不能同时启用——协调器有自己的 Worker 委托机制，fork 的"继承完整上下文"设计与协调器的"Worker 从零开始"哲学相矛盾。
8.3 协调器模式（Coordinator）

协调器模式由 COORDINATOR_MODE feature gate 控制，它将主 Agent 转变为纯编排者——只负责分析任务、分配 Worker、综合结果，永远不直接操作文件。

关键文件：src/coordinator/coordinatorMode.ts

协调器角色定义

协调器的系统提示词由 getCoordinatorSystemPrompt 函数生成，包含 6 个精心设计的部分。第一部分 Your Role 定义协调器职责，核心约束是直接指挥 Worker、综合结果、与用户沟通。第二部分 Your Tools 列出 Agent、SendMessage、TaskStop 三个工具，核心约束是不要用 Worker 去琐碎地汇报文件内容。第三部分 Workers 说明 Worker 的能力和工具集，要求 subagent_type 必须为 worker。第四部分 Task Workflow 规定四阶段工作流加并发管理，核心约束是"并行是你的超能力"。第五部分 Writing Worker Prompts 是提示词编写规范，核心约束是"永远不要写根据你的发现"。第六部分 Example Session 给出完整的多轮交互示例，覆盖从研究到修复的端到端流程。

协调器可用工具

协调器的工具集被严格限制——这是核心设计约束。四个工具分别是：第一，Agent，用来派生新 Worker；第二，SendMessage，用来继续已有 Worker，利用其已加载的上下文；第三，TaskStop，用来终止 Worker，是方向错误时的止损手段；第四，subscribe_pr_activity，用来订阅 GitHub PR 事件（若可用）。

协调器不能使用 Bash、Edit、Read 等工具——这确保它只做编排，不做执行。而 INTERNAL_WORKER_TOOLS 这一组，包括 TeamCreate、TeamDelete、SendMessage 和 SyntheticOutput，会从呈现给协调器、用来描述 Worker 可用工具的那份清单里剔除。这些工具是协调器和 Leader 内部编排用的，不该出现在给 Worker 的能力说明里。这四个里，真正落在 Worker 工具集内、会被实际过滤掉的只有 SyntheticOutput，其余三个本就不在集合里，属于防御性过滤。

为什么协调器不能执行？这不仅仅是分工问题——如果协调器既做决策又做执行，它会倾向于"自己动手比委托更快"，从而退化为一个普通的单 Agent。工具集的硬限制强制它必须通过 Worker 完成所有实际操作，这保证了任务分配的客观性和并行化。

Worker 工具集

Worker 根据模式获得不同的工具。（代码从略：这段代码来自 src/coordinator/coordinatorMode.ts。要点是当环境变量 CLAUDE_CODE_SIMPLE 为真时，Worker 只有 Bash、Read、Edit 三个工具，即简单模式；否则取 ASYNC_AGENT_ALLOWED_TOOLS 白名单的全部工具，再过滤掉 INTERNAL_WORKER_TOOLS 内部工具，即完整模式。）

具体来说：简单模式由 CLAUDE_CODE_SIMPLE 控制，工具为 Bash、Read、Edit；完整模式拥有 ASYNC_AGENT_ALLOWED_TOOLS 中的所有工具，但排除内部工具；MCP 工具自动可用；技能通过 SkillTool 委托。

Worker 工具上下文注入

getCoordinatorUserContext 做了一件看似简单但至关重要的事：它构建一个 workerToolsContext 字符串，注入到协调器的用户上下文中。这个字符串告诉协调器三件事。第一，Worker 有哪些工具。协调器要知道 Worker 的能力边界，才能写出可行的 prompt，不会去要求 Worker 用它根本没有的工具。第二，有哪些 MCP 服务器可用——如果连接了 Slack MCP，协调器就知道可以派 Worker 发消息。第三，Scratchpad 目录路径——如果启用了 Scratchpad，协调器可以指导 Worker 在共享目录中写入发现。

这是上下文工程在编排层面的体现——协调器不是在盲目委托，而是根据 Worker 的实际能力来制定可行的任务计划。

标准工作流

工作流流程描述：首先，用户请求由协调器分析任务、制定计划；然后，协调器派出三个 Worker 并行研究，再把所有发现综合起来，具体化为实施指令，下发给两个实施 Worker 和一个验证 Worker；最后，协调器汇总实施与验证的结果。

四个阶段的并发管理规则：第一是研究阶段，并发策略为自由并行，原因是只读操作，无冲突风险。第二是综合阶段，并发策略为协调器串行，原因是必须理解所有发现后才能下发指令。第三是实施阶段，并发策略为按文件集串行，原因是同一文件的写入必须串行化，防止冲突。第四是验证阶段，可以与不同文件区域的实施并行，原因是验证不修改被测代码。

协调器提示词设计精要

getCoordinatorSystemPrompt 中蕴含了多条经过实践验证的设计原则。

原则一，"Never write based on your findings"，永远不要写"根据你的发现"。协调器必须自己理解研究结果，然后写出包含具体文件路径、行号和修改内容的实施指令。"Based on your findings" 是将理解能力委托给 Worker，违背了协调器的核心职责。（代码从略：这段代码对比了两种写法。反模式是懒惰委托，只说"根据你的发现，修复认证 bug"；正确做法是综合后的具体指令，明确指出 src/auth/validate.ts 第 42 行的空指针问题，说明 Session 的 user 字段在会话过期但 token 仍在缓存时是 undefined，并要求在访问 user.id 前加一个空值检查。）

为什么这条规则如此重要？因为它定义了协调器的不可委托职责——综合理解。如果协调器只是转发消息（"Worker A 发现了一些东西，Worker B 你去处理"），它就退化成了一个消息路由器，没有任何智能编排的价值。强制协调器在综合阶段"理解并具体化"，是保持编排质量的关键。

原则二，"Every message you send is to the user"，你发出的每条消息都是对用户说的。这条规则防止协调器在长时间运行时保持沉默。Worker 的 task-notification 是内部信号，不是对话伙伴——协调器不应该回复通知，而应该向用户报告进展。

原则三，"Don't set the model parameter"，不要设置 model 参数。协调器提示词中明确要求不要为 Worker 设置 model 参数。原因是 Worker 默认使用与协调器相同的模型来处理实质性任务。如果协调器为了"节省成本"设置了更便宜的模型，Worker 在复杂实施任务中可能表现不佳——这是一个容易犯的错误。

原则四，"Add a purpose statement"，加上目的声明。协调器被要求在 Worker prompt 中包含"目的声明"——例如"This research will inform a PR description"，意思是这项研究将用于撰写 PR 描述。这是微妙但重要的提示工程：Worker 知道产出的用途后，会调整输出的深度和格式。为 PR 描述做的研究会更注重用户可见的变化，为 bug 修复做的研究会更注重根因分析。

原则五，Continue 与 Spawn 的决策。五种场景各有对策：研究探索了需要编辑的文件，选 Continue 继续，原因是 Worker 已有文件上下文；研究范围广但实施范围窄，选 Spawn 新建，原因是避免探索噪声，聚焦的上下文更干净；需要纠正失败或扩展最近的工作，选 Continue，原因是 Worker 有错误上下文；要验证其他 Worker 刚写的代码，选 Spawn，原因是验证者应以新鲜视角审视；上次实施方法完全错误，选 Spawn，原因是错误上下文会锚定重试思路。

最后一条特别有深意：当一个 Worker 的方法完全错误时，它的对话历史里塞满了错误假设和失败尝试。如果继续使用这个 Worker，模型倾向于基于已有上下文做小修小补，也就是"锚定效应"，而不是从根本上换一种方法。Spawn 一个全新的 Worker 可以避免这种认知锚定。

原则六，验证等于证明代码有效，不是确认代码存在。验证 Worker 必须做到四件事：运行测试并启用相关功能、调查类型检查错误而不轻易判定为"无关"、始终保持怀疑、独立复测。

原则七，Worker 看不到你的对话。每个 Worker 提示词必须是自包含的。协调器提示词中反复强调这一点，原话是 Workers can't see your conversation, every prompt must be self-contained，即 Worker 看不到你的对话，每个提示词都必须自包含。

这是初学者最容易犯的错——写出"请继续刚才的工作"这种 prompt，但 Worker 根本不知道"刚才"指什么。

8.4 Swarm 执行后端

Swarm 系统支持创建命名 Agent 团队，Agent 之间通过信箱对等通信。

关键文件：src/utils/swarm/backends/ 目录

三种后端

后端检测的流程描述：首先检测当前是否在 tmux 内，是就直接用 Tmux 后端。然后检测是否在 iTerm2 内：是的话再看 it2 命令行工具是否可用，可用就用 iTerm2 后端；不可用则看 tmux 是否可用，可用就回退到 Tmux，不可用就报错并给出安装指引。不在 iTerm2 内时，接着检测是否为非交互式环境：是就用 InProcess 后端；否则看 tmux 是否可用，可用用 Tmux，不可用则报错。

三种后端的对比：第一，Tmux 后端，实现方式是创建和管理 tmux 分屏面板，特点是支持隐藏和显示，最常用。第二，iTerm2 后端，通过 it2 命令行工具使用原生 iTerm2 面板，特点是 macOS 原生体验。第三，InProcess 后端，在同一个 Node.js 进程内运行，特点是靠 AsyncLocalStorage 隔离，共享 API 客户端和 MCP 连接。

后端选择优先级的设计考量

后端检测的优先级不是随意排列的，每一步都有明确的理由。

第一，已在 tmux 内，就直接用 Tmux。用户已经有了 tmux 分屏基础设施，在 tmux 内再起一个 tmux session 会造成嵌套混乱，沿用现成环境最自然。

第二，在 iTerm2 内、且 it2 命令行工具可用，就用 iTerm2。它给的是 macOS 原生的窗格体验，创建、分割的是 iTerm2 窗格而不是 tmux 面板；it2 不可用时回退到 tmux，因为 iTerm2 环境里 tmux 通常也在。

第三，非交互式环境用 InProcess。CI、CD、SDK 调用这类没有终端的场景创建不了可视化面板，InProcess 后端在同一进程内运行 Worker，是唯一可行的选择。

第四，其他情况退回到 tmux。前面几条都不满足时，tmux 是最后方案，它几乎在所有 Linux 和 macOS 系统上都能用。

统一接口

所有后端实现统一的 TeammateExecutor 接口。（代码从略：这个接口定义了五个方法。spawn 负责创建队友；sendMessage 向指定 Agent 发送消息；terminate 优雅关闭；kill 立即终止；isActive 检查 Agent 是否存活。）

terminate 和 kill 分工不同：terminate 发送优雅关闭请求，Agent 可以完成手头工作再退出；kill 通过 AbortController 立即中断。协调器在 Worker 方向错误时使用 TaskStop，它映射到 kill；在正常结束时使用 terminate。

InProcess 执行详解

InProcess 后端是最轻量的执行方式，适用于非交互式环境（如 CI、CD）。核心文件：src/utils/swarm/inProcessRunner.ts。

AsyncLocalStorage 上下文隔离：每个 Worker 通过 runWithTeammateContext 在独立的 AsyncLocalStorage 上下文中运行。Node.js 的 AsyncLocalStorage 提供了一种在异步调用链中传递上下文的机制——每个 Worker 的异步调用栈（Promise 链、回调等）都能访问自己的 TeammateIdentity，即使它们在同一个 Node.js 事件循环中交错执行。

示意图的朗读描述：Leader Agent 运行在主进程上下文中，经过 AsyncLocalStorage 上下文隔离层之后，Worker 1 和 Worker 2 各自拥有独立的 TeammateIdentity 和独立的 AbortController。同时，Leader 和两个 Worker 以虚线共享同一个 API 客户端和同一组 MCP 连接。

为什么 API 客户端和 MCP 连接可以共享？因为它们是无状态的连接复用——HTTP 客户端和 WebSocket 连接都线程安全，多个 Worker 可以并发使用同一个连接而不会干扰。这避免了为每个 Worker 建立独立连接的开销，包括 TCP 握手、TLS 协商、MCP 初始化等。

权限同步机制：Worker 执行工具时需要权限审批。InProcess 后端使用两种权限桥接方式。

第一种，Leader 桥接是首选：Worker 直接调用 Leader 的 ToolUseConfirm 对话框，界面上给这个 Worker 标一个 badge，让用户知道是谁在请求。权限确认直接在终端弹出，用户马上看到并做决定。

第二种，信箱通信是后备：Worker 用 writeToMailbox 把权限请求写进信箱，Leader 用 readMailbox 读取并响应，中间由 registerPermissionCallback 和 processMailboxPermissionResponse 衔接。它在 Leader 桥接不可用时顶上，比如 Leader 正忙着处理别的请求。

AbortController 独立性：每个 Worker 有独立的 AbortController。这意味着三点：一个 Worker 的失败不影响其他 Worker；协调器中断不级联到 Worker，Worker 可以被显式 TaskStop；killInProcessTeammate 通过 abort controller 立即终止特定 Worker。

Scratchpad：跨 Worker 知识共享

当 tengu_scratch feature gate 启用时，系统提供一个共享的 Scratchpad 目录。（代码从略：这段代码来自 src/coordinator/coordinatorMode.ts。要点是当 Scratchpad 目录存在且开关启用时，向提示词追加一段说明，告知 Scratchpad 目录路径，Worker 可以在这里无需权限提示地读写，用它存放持久的跨 Worker 知识。）

Workers 可以在这个目录中自由读写文件（无需权限确认），用于持久化跨 Worker 的知识——例如研究发现、中间结果、共享配置。

为什么需要 Scratchpad？没有它，Worker 之间只能通过协调器中转信息。这有两个问题：第一是延迟，Worker A 的发现必须先回传给协调器，协调器综合后再传给 Worker B——多了一个来回；第二是信息丢失，协调器综合时可能丢失细节（比如具体的行号），Worker B 拿到的是协调器的理解而非原始发现。

Scratchpad 提供了一个直接的旁路通道：Worker A 将详细发现写入文件，Worker B 直接读取——无需经过协调器的"理解和转述"。

8.5 Worker 结果传递

子 Agent、Worker 完成任务后，结果如何安全、可靠地回到父级？这涉及两条截然不同的返回路径、通知去重机制，以及针对 prompt injection 的安全分类。

同步与异步：两条返回路径

Worker 的结果传递分为同步和异步两条路径，它们的机制完全不同。

同步路径，由 agentToolUtils.ts 中的 finalizeAgentTool 函数实现：当子 Agent 同步执行时，父 Agent 阻塞等待。完成后，系统提取子 Agent 最后一条 assistant 消息的文本内容——不包含中间的工具调用过程——包装为 AgentToolResult，直接作为 tool_result 嵌入父级对话。（代码从略：这段代码是同步结果的结构。字段包括 status 为 completed；agentId；content 为最终结果文本；以及总工具调用次数、总耗时毫秒数和总 token 数。）

异步路径，由 LocalAgentTask.tsx 中的 enqueueAgentNotification 函数实现：异步 Agent 在后台运行，父 Agent 立即收到一个"已启动"的响应。当任务完成——无论成功、失败还是被终止——结果以 task-notification XML 格式作为 user 角色消息投递到父级的下一轮对话中。（代码从略：这段 XML 是一个 task-notification 示例。包含任务 ID；状态为 completed；摘要说明某个研究查询引擎的 Agent 已完成；result 节点放详细结果内容；usage 节点记录总 token 约 71330、工具调用 21 次、耗时约 81748 毫秒。）

关键字段有五个：task-id 是 Agent ID，可用于 SendMessage 继续该 Worker；status 取值为 completed、failed 或 killed；summary 是人类可读的结果摘要，取值形如 completed、failed 加错误信息，或 was stopped；result 是 Worker 的文本输出（可选），协调器据此做综合决策；usage 记录 token 使用量、工具调用次数和耗时，用于成本追踪。

task-notification 以 user 角色消息到达。协调器通过 task-notification 开头标签区分它们和真正的用户消息。这个设计选择是因为 Claude API 的消息格式要求——只有 user 角色的消息能由系统注入，而 task-notification 更像一个"系统事件"，算不上真正的用户输入。

通知去重与安全检查

去重机制：enqueueAgentNotification 使用一个原子的 notified 标志（LocalAgentTask.tsx）防止重复通知。如果 TaskStop 已经标记了任务为已通知，后续的完成通知会被静默丢弃。这防止了一个 Worker 被 stop 后又恰好自然完成时向协调器发送两条通知。

安全分类器：当 TRANSCRIPT_CLASSIFIER feature gate 启用时，classifyHandoffIfNeeded 函数（agentToolUtils.ts）在返回子 Agent 结果给父级之前，对子 Agent 的完整对话记录运行安全分类。这是一种纵深防御机制——防止攻击者通过精心构造的文件内容（如 README 中嵌入的 prompt injection）利用子 Agent 作为"跳板"，将恶意指令注入父级对话。如果分类器标记了结果，安全警告会被前置到结果文本中。

Worker 生命周期

生命周期流程描述：Worker 的一生分六步。首先 Spawn，创建 TeammateIdentity 和 AbortController；然后 Configure，构建工具集、设置权限桥接；接着 Build Prompt，用 getSystemPrompt 加 Worker 系统提示词构建提示词；再进入 runAgent 主循环，执行工具调用和流式输出；完成后根据结局发出通知——成功对应 status 为 completed，失败对应 failed，被停止对应 killed，都以 task-notification 形式发出；最后 Cleanup，调用 unregisterPermissionCallback、unregisterPerfettoAgent 和 evictTaskOutput 完成清理。

错误处理与恢复

Worker 失败时，协调器有多种恢复策略。四种场景各有对策：测试失败时，推荐用 SendMessage 继续同一个 Worker，因为 Worker 有完整的错误上下文；方法完全错误时，推荐 Spawn 新 Worker，避免错误上下文锚定重试思路；Worker 被 TaskStop 时，可以用 SendMessage 重新定向，因为被停止的 Worker 可以继续；多次纠正失败时，推荐报告给用户，因为可能需要人类判断。

协调器提示词中明确指出处理策略。（代码从略：这段提示词要求，当 Worker 报告失败时，用 SendMessage 继续同一个 Worker，因为它有完整的错误上下文；如果一次纠正失败，就换一种方法，或报告给用户。）
8.6 Plan 模式：两阶段执行

Plan 模式在 Agent 的工具调用循环中插入了一个审批关卡——进入 Plan 模式后，系统级剥离写入权限，Agent 只能读取代码和撰写计划文件；用户审批计划后，权限恢复，Agent 按计划执行修改。

关键文件：src/tools/EnterPlanModeTool/ 目录、src/tools/ExitPlanModeTool/ 目录、src/utils/planModeV2.ts 和 src/utils/plans.ts。

两阶段设计

流程描述：阶段一是只读探索。首先调用 EnterPlanMode 进入计划模式；然后用 Read、Grep、Glob 等只读工具探索代码，设计方案并写入计划文件；最后调用 ExitPlanMode。随后是审批关卡，由用户审批：拒绝则回到探索阶段。批准则进入阶段二，可写实施，按计划使用 Edit、Write、Bash 等工具执行修改。

两个阶段的对比：探索阶段使用 plan 权限模式，可写范围仅限计划文件，Agent 行为是使用只读工具加 Explore、Plan 子 Agent；实施阶段恢复原权限模式，可写范围是全部已授权工具，Agent 按审批通过的计划执行。

权限剥离与恢复

进入 Plan 模式时，prepareContextForPlanMode 函数（src/utils/permissions/permissionSetup.ts）会做一层精细的权限管理。它的真实语义和常见直觉有出入，有三点值得说清楚。

第一，只记旧模式、不在函数内切 plan：它把进入 Plan 前的权限模式（如 default 或 auto）记到 prePlanMode，供退出时恢复，并返回一个新的 context；函数体内从不把模式设为 plan。真正切到 plan 模式是在调用方 EnterPlanModeTool 里，通过 applyPermissionUpdate 的 setMode 更新完成的。

第二，危险权限的处置挂在"plan 里是否启用 auto"上，而不是"从 auto 进入就剥离"：从 auto 模式进入时，如果 plan-auto 开启（由 shouldPlanUseAutoMode 判断），就保持 auto、不剥离；如果关闭，则调用 restoreDangerousPermissions——是恢复而非剥离。真正调用 stripDangerousPermissionsForAutoMode 去剥离危险权限的，是"当前非 auto、且 plan-auto 开启"那条分支，目的是别让自动分类器在探索阶段批准写入操作。

第三，以上整套逻辑受 TRANSCRIPT_CLASSIFIER feature gate 控制；开关未启用时，函数只记下 prePlanMode 后把 context 原样返回。

被剥离的"危险权限"有三类：Bash 工具级别的 allow 规则，python:*、node:* 这类脚本解释器前缀，以及 agent(*) 这类 Agent 通配符。这些权限在用户审批计划后自动恢复。

设计决策：为什么不直接禁用所有写入工具？Plan 模式保留了一个可写表面——计划文件，存储在 ~/.claude/plans/ 目录下、以 slug 命名的 md 文件。Agent 要把探索发现和设计方案持久化到这个文件里，供用户审阅。"只允许写计划文件"这个设计在安全性和实用性之间取得平衡：既不动代码，又能产出一份可供审阅的方案。

Plan 模式的五阶段工作流

系统提示词（src/utils/messages.ts）为 Plan 模式定义了一个结构化的工作流：第一步，初步理解，使用 Explore 子 Agent 调查代码库；第二步，方案设计，使用 Plan 子 Agent 设计实现方案；第三步，方案审查，读取关键文件，确保方案可行；第四步，编写计划，将最终方案写入计划文件，这是唯一可编辑的文件；第五步，退出 Plan，调用 ExitPlanMode，触发用户审批。

审批与状态转换

流程描述：ExitPlanMode 被调用后，首先读取计划文件内容；然后判断执行上下文——如果是协调器的 Worker，就把 plan_approval_request 发送到团队领导信箱；如果是普通用户，就显示审批对话框。两条路都汇入审批结果：批准则恢复 prePlanMode 记录的原模式、恢复被剥离的权限，并把计划内容注入上下文；拒绝则继续留在 Plan 模式，根据反馈修改方案。

审批通过后，计划内容作为 tool_result 注入对话，确保模型在实施阶段能引用具体方案。

为什么需要两阶段设计？

传统的 Agent 执行模式是"边想边做"——模型一边分析问题一边修改代码。这在简单任务中效率很高，但在复杂任务中会导致三类问题。一是方向性返工：Agent 在只看了局部代码后就动手修改，后续发现整体方向不对，已有修改全部作废。二是无计划的局部修改：缺少全局视角的逐文件修改可能引入不一致，尤其在大型重构中。三是审批粒度过细：用户被迫逐个工具调用地审批，无法看到全貌就要做决定。

两阶段设计通过一个审批关卡强制 Agent "先想清楚再动手"。源码中的关键约束是系统提示词中的这句话，大意是：用户已表明还不希望你执行，你绝不能做任何编辑、不能运行任何非只读工具、也不能对系统做任何更改。

这不是建议，是硬约束——Plan 模式下写入工具的权限被系统级剥离，即使模型尝试调用也会被拒绝。
8.7 设计洞察

十条设计洞察。第一，协调器不执行是核心约束：防止协调器既做决策又做执行，保证任务分配的客观性。这也是为什么协调器的工具集被严格限制为 Agent、SendMessage、TaskStop 三个工具。

第二，"Never write based on your findings" 是最重要的提示词设计：强制协调器综合理解研究结果，而非将理解委托给 Worker。这个约束将协调器从消息转发器提升为真正的智能编排者。

第三，Continue 与 Spawn 不是默认选择，取决于上下文重叠度：重叠度高就继续，重叠度低就新建。这个决策框架避免了无脑复用或无脑新建。

第四，AbortController 独立性保证故障隔离：一个 Worker 的崩溃不会连锁影响其他 Worker。这是并行系统的基本可靠性要求。

第五，后端检测优先级考虑用户环境：优先级依次是 Tmux、iTerm2、InProcess，最大化利用已有终端能力。

第六，Scratchpad 解决跨 Worker 知识共享：没有它，Worker 之间只能通过协调器中转信息，增加延迟和信息丢失风险。

第七，Plan 模式的审批关卡是信任的物化：两阶段设计不只是 UX 改进——它把"用户信任"从隐性变成显性。过去是每次工具调用都弹一次权限窗，现在是一次性审批整体方案。这在团队协作里尤其重要：协调器 Worker 的计划要经过团队领导审批，而不是每改一个文件都确认一次。

第八，Fork 是伪装成架构模式的缓存优化：Fork 子 Agent 的核心动机不是"继承上下文"——而是让多个子级共享父级的 Prompt Cache。CacheSafeParams 类型明确要求"字节级相同"就是最好的证据。继承上下文是缓存共享的副产品，不是设计目标。

第九，上下文隔离默认最大安全：createSubagentContext 将所有可变状态默认设为隔离，即空操作或克隆，开发者必须通过 shareSetAppState、shareAbortController 等参数显式开启共享。这种 deny by default 设计意味着新增的子 Agent 功能天生是安全的——除非开发者有意识地打开共享。

第十，工具过滤实现纵深防御：四层过滤依次是全局禁止、自定义限制、异步白名单、类型级禁止，确保即使某一层有 bug，其他层仍能拦截危险工具访问。MCP 工具的"始终放行"看似是例外，实际上是信任边界的正确划分——用户配置的外部工具由用户自己负责安全性。

动手实践：在 claude-code-from-scratch 项目中，Agent 主循环（src/agent.ts）实现了基础的工具调用循环。尝试在此基础上增加一个简单的"plan 模式"——在执行工具前先收集所有计划的操作，让用户一次性审批。

上一章：第 7 章 Hooks 与可扩展性。下一章：第 9 章 Plan 模式。
