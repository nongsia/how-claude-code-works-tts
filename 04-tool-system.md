---
title: 第 4 章：工具系统（朗读版）
---

第 4 章：工具系统

本文是 docs/04-tool-system.md 的朗读版，表格与代码已转为口语描述，内容未增删。

工具系统是 Claude Code 能力的载体：60 多个内置工具，加上 MCP 接进来的外部世界。

Claude Code 的所有能力，包括文件读写、Shell 命令、代码搜索、子 Agent 派生、MCP 外部服务调用，都通过统一的工具系统暴露给模型。模型不直接操作文件系统或网络，而是通过调用工具来完成一切副作用操作。工具系统是连接“模型智能”与“真实世界”的唯一桥梁。

这套系统的核心架构分为三层。第一层是设计层，也就是 src/Tool.ts 里的 Tool 泛型接口，它定义每个工具必须实现的契约：执行逻辑、输入 Schema、只读、破坏性、并发安全这三类安全语义标记、权限检查，以及 UI 渲染。第二层是组装层，也就是 src/tools.ts 里 getAllBaseTools、getTools、assembleToolPool 的三级接力，从编译时裁剪到运行时过滤，最终将内置工具和 MCP 工具合并为统一的工具池。第三层是执行层，也就是 src/services/tools 目录下的 StreamingToolExecutor，它在模型流式输出的同时并发执行工具，处理权限检查、Hook 回调和结果格式化。

这种设计带来两个关键优势。第一，新增工具只需实现 Tool 接口，无需修改执行流水线或权限系统。第二，安全语义 isReadOnly 和 isDestructive 编码为接口方法而非外部配置，确保安全属性与工具实现始终同步。

本章路线图：4.1 到 4.2 介绍接口定义与组装流水线；4.3 列出内置工具全景；4.4 到 4.5 讲解执行生命周期与并发控制；4.6 到 4.7 深入分析最复杂的两个工具，也就是 BashTool 和 AgentTool；4.8 到 4.10 覆盖大结果处理、MCP 集成和延迟加载；4.11 到 4.12 总结设计洞察与 UI 渲染模式。

4.1 Tool 接口定义

上述三层架构的起点是 src/Tool.ts 里的 Tool 接口，它是内置、MCP、REPL 所有工具的统一契约，也是整个系统的核心类型。

（代码从略：这段代码定义了 Tool 泛型接口。元数据部分包括工具唯一标识 name、兼容旧名称的 aliases、结果最大字符数 maxResultSizeChars，以及表示是否延迟加载的 shouldDefer。核心执行部分是 call 方法，返回 Promise 形式的 ToolResult。提示词部分包括生成描述的 description 和生成提示词的 prompt。Schema 部分包括 Zod 输入 Schema inputSchema，以及供 API 兼容使用的 JSON Schema。安全与权限部分包括判断是否可并发执行的 isConcurrencySafe、判断是否只读的 isReadOnly、判断是否破坏性操作的 isDestructive、校验输入的 validateInput，以及检查权限的 checkPermissions。UI 渲染部分是两个 React 组件方法，分别渲染工具调用消息和工具结果消息。）

每个工具返回的 ToolResult 不仅包含数据，还可以注入额外消息或修改上下文。

（代码从略：ToolResult 类型包含三个字段。data 是工具输出数据；newMessages 是额外注入的消息；contextModifier 是上下文修改器函数。）

buildTool 工厂模式

所有工具的创建都通过 buildTool 工厂函数完成。这个函数将 TOOL_DEFAULTS 与工具的自定义定义合并，确保每个工具都有完整的方法集。

（代码从略：这段代码定义了 TOOL_DEFAULTS 默认值和 buildTool 工厂函数。默认值包括：isEnabled 默认返回 true；isConcurrencySafe 默认返回 false，防止并发问题；isReadOnly 默认返回 false，意味着需要权限检查；isDestructive 默认返回 false；checkPermissions 默认允许；toAutoClassifierInput 默认返回空字符串，跳过分类器。buildTool 把这些默认值与工具自定义定义合并，并让 userFacingName 默认返回工具名。）

这是一套 fail-closed，也就是默认关闭的安全设计。第一，isConcurrencySafe 默认返回 false，新工具默认不可并发执行。只有经过验证确实安全的工具，比如纯读取操作，才显式声明为 true。这避免了新增工具因遗漏并发安全标记而导致竞态条件。第二，isReadOnly 默认返回 false，默认假设工具有写入副作用，因此必须经过权限检查。只读工具如 GrepTool、GlobTool 会显式声明自己为只读，以跳过权限弹窗。第三，toAutoClassifierInput 默认返回空字符串，默认跳过 LLM 分类器的自动审批。这意味着安全相关的工具不会被意外自动批准，必须由工具作者显式提供分类器输入格式。

这种设计确保了：任何新工具在缺少显式配置的情况下，都会走最保守的路径，需要权限、不可并发、不自动批准。

工具目录结构

每个工具独立存放在 src/tools 下的同名目录中，遵循统一的文件组织约定。

以 FileEditTool 目录为例，这个结构包含六个文件。FileEditTool.ts 是主实现，包含 call、validateInput 和 checkPermissions 方法。UI.tsx 负责 React 渲染，实现 renderToolUseMessage 和 renderToolResultMessage。types.ts 定义 Zod 输入 Schema 和 TypeScript 类型。prompt.ts 存放工具特定的 system prompt 注入内容。constants.ts 定义常量。utils.ts 是辅助函数，比如 diff 生成和引号标准化。

这种分离的好处有三点。第一，执行逻辑在 ts 文件、渲染逻辑在 UI.tsx，二者完全解耦，修改 UI 不影响工具行为。第二，types.ts 中定义的 Zod Schema 既用于运行时验证，也自动转换为 JSON Schema 发送给 API。第三，每个工具可以通过 prompt.ts 向系统提示词注入工具特定的使用指南，例如 FileEditTool 注入关于精确匹配的规则。

4.2 工具注册与组装

src/tools.ts 定义了工具从定义到可用的三层组装流水线，依次完成编译时裁剪、运行时过滤和缓存感知排序。

首先，第一层 getAllBaseTools 通过静态 import 导入约 31 个核心工具，并条件加载约 20 个工具，另外还有 3 个用 lazy require 加载。然后，第二层 getTools 基于权限上下文过滤。最后，第三层 assembleToolPool 把内置工具与 MCP 桥接工具合并并做去重处理，形成最终工具池。

第一层：getAllBaseTools，编译时工具裁剪

getAllBaseTools 位于 src/tools.ts 的 193 到 251 行，是所有工具的单一事实来源，返回当前构建环境下所有可能可用的工具。

核心工具约 31 个，通过静态 import 直接导入，始终存在。全部工具枚举下来约 55 到 60 个。

（代码从略：这段代码用静态 import 导入 BashTool、FileReadTool、FileEditTool 等核心工具。）

Feature-gated 工具约 20 个，通过条件 require 加载。另有 3 个 Team 和 SendMessage 工具也用 lazy require，但那是为了打破循环依赖，不属于 feature gate。

（代码从略：这段代码展示了条件加载的写法。当 PROACTIVE 或 KAIROS 特性开启时才通过 require 加载 SleepTool，当 HISTORY_SNIP 特性开启时才加载 SnipTool，否则它们的值就是 null。）

这里的 feature 不是运行时函数，它是 Bun 打包器的编译时宏。构建面向外部用户的版本时，feature('PROACTIVE') 在编译阶段求值为 false，整个三元表达式简化为 const SleepTool = null，require 调用则被死代码消除，也就是 Dead Code Elimination，物理删除。这意味着内部工具不只是被隐藏，它们在外部构建的二进制文件中根本不存在，从根本上杜绝了通过运行时手段绕过 Feature Gate 的可能。

还有一个优化：当 hasEmbeddedSearchTools 返回 true 时，也就是 Anthropic 内部构建将 bfs 和 ugrep 编译进了 Bun 二进制文件，GlobTool 和 GrepTool 会被排除。因为 shell 别名已经指向了更快的嵌入式实现，专用工具就没有必要了。

第二层：getTools，运行时上下文过滤

getTools 位于 src/tools.ts 的 271 到 327 行，在运行时根据当前环境和权限上下文过滤工具。它包含四层递进过滤。

第一层是 SIMPLE 模式，由 CLAUDE_CODE_SIMPLE 环境变量或 --bare 标志开启。它将工具集削减到最小核心，仅保留 BashTool、FileReadTool、FileEditTool。这是最轻量的工具配置，适用于资源受限或嵌入式场景。当 REPL 模式同时启用时，REPLTool 会顶替这三个工具，因为 REPL 的 VM 内部已经封装了它们。

第二层是 REPL 模式过滤。当 isReplModeEnabled 为 true 且 REPLTool 可用时，REPL_ONLY_TOOLS 集合中的工具，包括 Bash、FileRead、FileEdit 等，会从直接工具列表中隐藏。这些工具仍然存在于 REPL VM 的执行上下文中，但模型不能直接调用它们，必须通过 REPL 工具间接使用。

第三层是 Deny 规则过滤。filterToolsByDenyRules 检查每个工具是否匹配全局 deny 规则。一个没有 ruleContent 的 deny 规则，也就是 blanket deny，会完全移除对应工具，使模型在 system prompt 中根本看不到它。对于 MCP 工具，前缀匹配规则如 mcp__server 会一次性移除该服务器的所有工具。这是在模型看到工具列表之前就完成的，而不是在调用时才检查。

第四层是 isEnabled 运行时检查。系统调用每个工具的 isEnabled 方法，过滤掉返回 false 的工具。这允许工具根据运行时条件，比如依赖是否可用，自行决定是否启用。

第三层：assembleToolPool，合并与缓存感知排序

assembleToolPool 位于 src/tools.ts 的 345 到 367 行，是最终的组装点，将内置工具和 MCP 工具合并为统一的工具池。

（代码从略：这个函数先取出内置工具和通过 deny 规则过滤后的 MCP 工具，然后按名称做分区排序，内置工具作为连续前缀，MCP 工具作为后缀，最后按名称去重，返回统一的工具池。）

这段代码有两个关键设计决策。

第一个是分区排序而非全局排序。内置工具按字母排序形成一个连续的前缀块，MCP 工具按字母排序后追加为后缀块。为什么不直接对所有工具做一次全局排序？因为 API 服务器的缓存策略，也就是 claude_code_system_cache_policy，在最后一个内置工具之后设置了缓存断点。如果做全局排序，一个名为 mcp__github__create_issue 的 MCP 工具会插入到 GlobTool 和 GrepTool 之间，导致所有下游缓存键失效。分区排序确保添加或移除 MCP 工具只影响后缀部分，内置工具前缀块更大，其缓存始终命中。

第二个是 uniqBy 按名称去重时内置优先。当内置工具和 MCP 工具同名时，uniqBy 保留首次出现的，也就是内置工具，因为内置工具在拼接数组中排在前面。这确保了 MCP 工具不会意外覆盖内置工具。

4.3 内置工具清单

Claude Code 包含 60 多个内置工具，按功能域分为 6 类。这些分类反映了 coding agent 的核心能力模型：文件操作是基础，读写搜索是最高频操作；Agent 管理与团队协作支撑多 Agent 执行；用户交互和系统控制保证人在回路中的控制力；工具扩展则把技能、延迟工具加载和 MCP、LSP 外部能力统一纳入扩展出口。工具集按覆盖开发者日常工作流 95% 的场景来选，剩下 5% 由 BashTool 这个万能后备加上技能和 MCP 扩展兜底。

第一类是文件操作，共 7 个工具。BashTool 负责执行 Shell 命令，是最复杂的工具。FileReadTool 负责读取文件内容，支持图片、PDF 和 Jupyter。FileEditTool 负责精确字符串替换编辑，是核心编辑工具。FileWriteTool 负责创建或覆盖文件。GlobTool 按模式匹配文件。GrepTool 用正则搜索文件内容，基于 ripgrep。NotebookEditTool 负责 Jupyter Notebook 编辑。

第二类是网络，共 2 个工具。WebFetchTool 获取网页内容。WebSearchTool 提供 API 驱动的网络搜索。

第三类是 Agent 管理与团队协作，共 8 个工具。AgentTool 派生子 Agent，是多 Agent 架构的核心。TaskOutputTool 输出任务结果。TaskStopTool 停止后台任务。TaskCreate、TaskGet、TaskUpdate、TaskList 这一组工具负责任务管理 v2，详见第 11 章。SendMessageTool 负责 Agent 间通信。TeamCreateTool 创建 Agent 团队。TeamDeleteTool 删除 Agent 团队。ListPeersTool 列出同级 Agent。

第四类是用户交互，共 3 个工具。AskUserQuestionTool 向用户提问。TodoWriteTool 管理待办列表。BriefTool 的模型侧名称是 SendUserMessage，负责向用户发送消息，它是助手模式和后台模式下用户可见的回复通道，支持 markdown 与附件。

第五类是系统，共 5 个工具。EnterPlanModeTool 进入规划模式。ExitPlanModeTool 退出规划模式。EnterWorktreeTool 进入 Git Worktree 隔离。ExitWorktreeTool 退出 Worktree。ConfigTool 负责配置管理。

第六类是工具扩展，共 6 个工具。SkillTool 加载并执行技能。ToolSearchTool 搜索并加载延迟工具。ListMcpResourcesTool 列出 MCP 资源。ReadMcpResourceTool 读取 MCP 资源。MCPTool 是 MCP 工具代理。LSPTool 负责语言服务器操作。

4.4 工具执行生命周期

当模型在流式输出中产生一个 tool_use block 时，这个调用请求并不会直接执行，它需要经历一条完整的处理流水线。从工具查找、输入验证、权限检查，其中可能涉及用户交互，到实际执行、结果格式化，再到 Hook 回调，共 8 个阶段。这条流水线对所有工具，包括内置、MCP、REPL，完全相同，是 4.1 节统一 Tool 接口的直接体现。理解这条流水线是理解 Claude Code 安全模型和执行语义的关键。

首先，模型输出 tool_use block 后进入工具查找和输入验证，系统按 name 和别名查找工具并检查废弃别名，再用 Zod Schema 解析强制和 validateInput 检查来验证输入。然后是并行启动阶段，Pre-Tool Hook 和 Bash 分类器同时运行，分类器做投机执行；随后进入权限检查，依次经过规则匹配、分类器自动审批、Hook 覆盖和交互式确认。接着是工具执行阶段，调用工具本身，产生流式进度事件，并处理超时和沙箱。最后是结果处理、Post-Tool Hook 和消息发射：结果经转换后大结果持久化到磁盘，成功时触发 PostToolUse、失败时触发 PostToolUseFailure，最终产出 tool_result block。

各阶段详解

第一阶段，工具查找。系统按 name 和 aliases 匹配工具定义。如果工具是通过已废弃的别名调用的，系统会在 tool_result 中附加一条废弃警告，引导模型在后续调用中使用新名称。如果未找到匹配工具，直接返回错误消息，这在模型幻觉出不存在的工具时会发生。

第二阶段，输入验证。输入验证分为两个阶段。（代码从略：这段代码展示了两个验证阶段。第一阶段是 Zod Schema 强制转换，safeParse 不仅验证，还会做类型强制，比如把字符串形式的数字转成数字，如果 Schema 验证失败，就把格式化的错误信息返回给模型。第二阶段是业务逻辑验证，仅在 Schema 验证通过后执行，调用 validateInput 并返回三种结果之一：验证通过、直接拒绝并附带消息和错误码，或者显示 UI 提示让用户决定。）

两阶段分离，Schema 层做结构验证，管字段存在性和类型；业务层做语义验证，如 FileEditTool 检查文件是否存在、FileWriteTool 检查是否已读过文件再写入。ask 行为模式允许工具在不确定的情况下把决策权交给用户，而非直接拒绝。

第三阶段，并行启动。Pre-Tool Hook 和 Bash 分类器同时启动，而不是串行等待。这两个操作可能各需要数十到数百毫秒，并行启动后总等待只取决于较慢的那一个，而不是两段相加。其中，Pre-Tool Hook 执行用户在 PreToolUse 事件下配置的外部脚本，可以返回 allow、deny 或者不干预。Bash 分类器对 BashTool 调用做投机性安全分类，判断命令是否只读，结果缓存以供权限检查使用。

第四阶段，权限检查。权限检查是整个流水线中最复杂的阶段，实现在 checkPermissionsAndCallTool，位于 src/services/tools/toolExecution.ts。它涉及多个决策源，按优先级链式求值，一旦某个环节做出明确决定，后续检查就不再执行。

第一个决策源是 Hook 权限覆盖，优先级最高。如果第三阶段的 Pre-Tool Hook 返回了权限决定，也就是 hookPermissionResult，它直接覆盖所有后续检查。Hook 可以返回三种结果。allow 表示跳过所有权限检查直接执行，例如企业内部 Hook 自动批准特定命令。deny 表示立即拒绝并附带拒绝原因。无决策则穿透到下一层检查。

第二个决策源是工具自身的 checkPermissions。每个工具可以实现自己的权限逻辑。大部分工具使用默认实现，直接返回 allow；BashTool 的 bashToolHasPermission 却是一个 200 多行的复杂实现，详见 4.6 节。文件工具会检查路径是否在允许的工作目录范围内。

第三个决策源是规则匹配。系统从 8 个来源收集权限规则。第一个来源是 userSettings，即用户主目录下的 ~/.claude/settings.json，属于用户级全局设置。第二个来源是 projectSettings，即项目里的 .claude/settings.json，属于项目级共享设置。第三个来源是 localSettings，即 .claude/settings.local.json，是不提交到 git 的个人设置。第四个来源是 flagSettings，即通过 --settings 参数传入的设置文件，属于命令行显式指定的设置。第五个来源是 policySettings，即组织策略，来自 managed-settings.json 或 API 下发的远程设置，是企业管理员下发的强制规则。第六个来源是 cliArg，即命令行参数指定的规则，例如用 --allowedTools 参数指定 Bash(git *)。第七个来源是 command，即斜杠命令或命令 frontmatter 附带的规则，例如命令定义里声明的 allowedTools。第八个来源是 session，即当前会话中用户的临时授权，在用户点击“允许一次”时生成。

每条规则有三种行为：allow 自动批准、deny 自动拒绝、ask 要求交互确认。这些来源之间并不是简单的线性优先级，真正决定结果的是行为优先级，deny 高于 ask，ask 高于 allow。跨来源的 deny 一律先生效，session 的临时授权盖不掉企业 policySettings 里的 deny。设置类来源彼此之间遵循后者覆盖前者，顺序是 userSettings、projectSettings、localSettings、flagSettings、policySettings，企业策略最强。规则的 ruleContent 支持三种匹配模式。第一是精确匹配，Bash(npm install) 仅匹配完全相同的命令。第二是前缀匹配，Bash(git commit:*) 匹配所有以 git commit 开头的命令。第三是通配符匹配，Bash(git *) 匹配所有以 git 加空格开头的命令。

第四个决策源是投机分类器结果，Bash 专用。在第三阶段中并行启动的 Bash 分类器此时返回结果。分类器是一个基于语义描述的 LLM 侧查询，用于判断命令是否匹配用户定义的 allow 或 deny 描述。如果分类器以高置信度判定命令安全，即匹配 allow 描述，权限弹窗就自动跳过。分类器的结果通过 pendingClassifierCheck 这个 Promise 异步传递，UI 层可以在展示弹窗前等待它。

第五个决策源是交互式确认弹窗。如果以上所有检查都没有做出明确决定，既不是自动允许也不是自动拒绝，用户会看到一个权限确认弹窗。弹窗包含四部分内容。第一是工具名称和完整输入，比如 Bash 命令文本。第二是破坏性命令警告，如果适用，来自 getDestructiveCommandWarning。第三是建议的权限规则，比如 Bash(git commit:*)，用户可以选择保存以避免未来重复确认。第四是三个操作选项：仅放行本次的“允许一次”、保存为规则的“始终允许”，以及“拒绝”。

第六个决策源是拒绝追踪。DenialTrackingState 跟踪连续权限拒绝。当同一工具或命令模式被多次拒绝后，系统会向对话中注入引导提示，帮助模型理解它应该尝试不同的方法，而不是反复请求已遭拒绝的操作。这防止了模型陷入请求权限、遭拒、再次请求的死循环。

第五阶段，工具执行。tool.call 执行实际操作。关键机制是 onProgress 回调，它允许工具在执行过程中实时发射进度消息。例如，BashTool 通过此回调流式传输 stdout 和 stderr 输出，用户可以实时看到命令的输出而不必等待命令完成。后台任务，即 run_in_background 为 true 的调用，在超时阈值后自动转为异步执行。

第六阶段，结果处理。mapToolResultToToolResultBlockParam 将工具的内部 ToolResult 转换为 API 兼容的 ToolResultBlockParam 格式。其中最要紧的是大结果处理：如果结果超过 maxResultSizeChars，完整内容保存到磁盘，模型接收到的是文件路径加截断指示符，详见 4.8 节。这避免了一次 grep 搜索结果炸掉整个上下文窗口。

第七阶段，Post-Tool Hook。Post-Tool Hook 分为两个独立事件。第一个是 PostToolUse，在工具执行成功时触发，脚本接收工具名称、输入和输出。第二个是 PostToolUseFailure，在工具执行失败时触发，脚本接收工具名称、输入和错误详情。两者是独立的 Hook 事件，用户可以分别配置不同的处理逻辑。例如，可以在 PostToolUse 中对 BashTool 的 git push 命令发送通知，在 PostToolUseFailure 中记录失败日志。需要注意，事件名在 settings 配置里大小写敏感、按字面匹配，必须写成 PostToolUse 和 PostToolUseFailure，写成小写不会触发。

错误处理与传播

工具执行流水线的错误处理遵循一个核心哲学：错误是数据，不是异常。在任何阶段发生的错误都不会导致进程崩溃或对话中断，它们一律转成带 is_error 为 true 标记的 tool_result 消息返回给模型，让模型可以自我纠正。

各阶段的错误形式如下。第一，Schema 验证阶段，错误类型是 Zod parse 失败，比如类型错误或缺失字段，处理方式是用 formatZodValidationError 格式化后包裹在 tool_use_error 的 XML 标签中。第二，业务验证阶段，validateInput 返回结果为 false，处理方式是返回验证错误消息并附带 errorCode。第三，权限拒绝阶段，用户点击拒绝或规则匹配 deny，处理方式是返回 CANCEL_MESSAGE 或具体的拒绝原因。第四，工具执行阶段，运行时异常，比如文件不存在、命令失败等，处理方式是 try-catch 捕获，由 classifyToolError 分类后记录遥测。第五，MCP 工具阶段，错误类型是 McpToolCallError 或 McpAuthError，Auth 错误触发 OAuth 流程，其他错误返回给模型。

classifyToolError 位于 src/services/tools/toolExecution.ts，它要解决一个问题：minified 构建中，JavaScript 的 error.constructor.name 会被混淆成 nJT 之类的短标识符，无法用于遥测分析。因此该函数采用了一个鲁棒的分类优先级链。第一，如果是 TelemetrySafeError 实例，使用其 telemetryMessage，也就是开发者显式标记为遥测安全的消息。第二，标准 Error 加 errno 码，返回 Error:ENOENT、Error:EACCES 等，这些是 Node.js 文件系统错误的稳定标识。第三，Error 实例且 name 长度大于 3，说明未被 minify，使用原始错误名。第四，其他 Error 返回 Error。第五，非 Error 值返回 UnknownError。

这种设计确保了遥测数据在任何构建模式下都是可分析的，同时避免泄漏文件路径或代码片段到遥测系统中。

4.5 并发控制

工具的并发执行遵循严格的规则。第一，并发安全的工具可并行：isConcurrencySafe 返回 true 的工具，如 FileReadTool、GrepTool、GlobTool，可以同时执行。第二，非并发安全的工具串行：isConcurrencySafe 返回 false 的工具，如 FileEditTool 和 BashTool 的写命令，必须独占执行。第三，isReadOnly 只是直觉：只读通常意味着并发安全，但二者是接口里两个独立方法，真正门控并发调度的是 isConcurrencySafe，不是 isReadOnly。

工具编排由 src/services/tools/toolOrchestration.ts 的 runTools 函数管理。它不是把所有只读工具一次性 filter 出来做整体 Promise.all，而是按输出顺序把连续的并发安全调用聚成一批并行执行，遇到非并发安全调用则单独成批串行。

（代码从略：这段代码是 partitionToolCalls 的核心，按 isConcurrencySafe 分批并保持原始顺序。它遍历每个工具调用，先用输入 Schema 解析输入，再调用 isConcurrencySafe 判断，判据是并发安全而不是只读。连续的并发安全调用并入同一批，否则新开一批。每个并发安全批次内部用 runToolsConcurrently 并行执行，并受并发上限 getMaxToolUseConcurrency 限流，默认是 10；非安全批次串行执行。）

StreamingToolExecutor：流式并行执行

上述静态编排策略有一个局限：它必须等待模型完整输出所有 tool_use blocks 后才开始执行。而实际上，模型的流式输出需要 5 到 30 秒，一个 tool_use block 可能在流式输出的前几秒就已完整，何必等到最后？

StreamingToolExecutor 位于 src/services/tools/StreamingToolExecutor.ts，约 530 行，正是为此设计。它在模型流式输出的同时，一旦检测到完整的 tool_use block，就立即启动执行。

（代码从略：这段代码定义了工具在 StreamingToolExecutor 中经历的 4 种状态，分别是 queued 排队、executing 执行中、completed 已完成和 yielded 已产出。同时定义了每个工具的跟踪信息 TrackedTool，包含 id、工具调用块、所属的 assistant 消息、是否并发安全，以及可选的结果消息。其中 pendingProgress 进度消息即时发射，不等待最终结果。）

并发控制规则直接嵌入执行器内部。

（代码从略：这是执行器私有的 canExecuteTool 方法，用于判断能否执行一个工具。当正在执行的工具数量为零时返回 true；否则只有新工具自身是并发安全，且所有正在执行的工具都是并发安全时，才返回 true。）

规则只有一条：当前没有工具在执行时，任何工具都可以启动；有工具在执行时，只有自身和所有正在执行的工具都标记为并发安全，新工具才能启动。非并发安全的工具必须独占执行。

时间线对比展示了这种优化的效果。串行执行是先等待所有 tool_use 输出完成，再依次执行 tool1、tool2、tool3。流式并行执行则是第一个 tool_use 完成后立即启动 tool1，随后 tool2、tool3 依次跟进，利用 5 到 30 秒的流式窗口覆盖工具延迟，结果在 API 输出完成时已经就绪。

典型场景下，工具执行延迟约 1 秒，而模型流式输出持续 5 到 30 秒。这意味着大部分工具执行可以完全隐藏在流式窗口内，用户感知的总延迟接近于纯 API 调用时间。

另一个关键设计是 progressAvailableResolve 唤醒信号。结果消费者 getRemainingResults 以事件驱动方式工作，当新结果或进度就绪时，执行器通过 resolve Promise 唤醒消费者，避免轮询开销。pendingProgress 消息，比如 BashTool 的 stdout 流，立即发射给 UI，不必等待工具最终完成。

分区算法与并发上限

在 StreamingToolExecutor 内部，工具分两类执行。第一类是并发安全工具，isConcurrencySafe 为 true，可以与其他并发安全工具同时执行，典型例子是 FileReadTool、GrepTool、GlobTool，它们只读取数据，不会互相干扰。第二类是非并发安全工具，必须独占执行，在它运行期间不能有其他工具同时执行，FileEditTool 和 BashTool 的写操作属于此类。

执行器的调度逻辑基于一个简单规则：当前没有工具在执行时，任何工具都可以启动；当有工具在执行时，新工具只能在自身和所有正在执行的工具都是并发安全的条件下启动。一旦遇到非并发安全工具，队列处理暂停，等待当前所有执行中的工具完成后，非并发安全工具独占运行。

并发数的硬性上限出现在静态编排路径上。toolOrchestration.ts 的 runToolsConcurrently 在并发批次上以 getMaxToolUseConcurrency 限流，默认是 10，可通过 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY 环境变量覆盖，防止模型一次发出 20 个 FileReadTool 调用时导致文件句柄耗尽或 I/O 竞争。而 StreamingToolExecutor 本身没有数字上限，它只靠 canExecuteTool 的并发安全性门控，可以同时启动任意多个并发安全工具；并不存在名为 MAX_TOOL_USE_CONCURRENCY 的常量。

此外，StreamingToolExecutor 还实现了兄弟取消机制，也就是 siblingAbortController。当一个 Bash 工具执行出错时，它会取消同批次中其他正在执行的工具。这避免了第一个命令失败但后续命令继续执行的问题。非 Bash 工具，如 FileReadTool、WebFetchTool，的错误则不会级联，这类失败通常各自独立，不影响兄弟工具。

结果的发射顺序始终与工具在模型输出中的出现顺序一致，也就是 FIFO，即使后面的工具先完成。这确保了消息流的确定性和可预测性。

4.6 BashTool 深度解析

BashTool 是整个工具系统中最复杂的工具，它的实现分布在 src/tools/BashTool 目录的 18 个源文件中，涵盖安全验证、权限管理、沙箱隔离、命令语义解析等多个子系统。这种复杂性源于一个根本矛盾：Shell 几乎可以做任何事，是最强大的工具；一条恶意命令就能删除整个文件系统，它又是最危险的工具。

（代码从略：这是 BashTool 的输入 Schema，包含五个字段。command 是 Shell 命令；timeout 是以毫秒计的超时；description 是用于 UI 展示的活动描述；run_in_background 表示是否异步执行；dangerouslyDisableSandbox 表示是否禁用沙箱。）

4.6.1 安全验证（bashSecurity.ts）

安全验证是 BashTool 的第一道防线，在权限检查之前执行。bashSecurity.ts 约 2600 行代码，实现了 23 个命名安全检查，通过 BASH_SECURITY_CHECK_IDS 常量映射为数字 ID，避免在遥测日志中记录可变字符串。

（代码从略：这段代码定义了 BASH_SECURITY_CHECK_IDS 常量，把 23 个安全检查依次映射为数字 1 到 23。它们分别是：不完整的命令片段；jq 的 system 函数调用；jq 文件参数限制；混淆的命令标志；Shell 元字符；管道或重定向中的危险变量；未引用内容中的换行符；命令替换；输入重定向；输出重定向；IFS 变量注入；git commit 中的替换；访问 proc self environ 进程环境；畸形 token 注入；反斜杠转义空白；花括号展开；控制字符；Unicode 空白同形字；词中井号注释攻击；Zsh 危险命令；反斜杠转义运算符；注释与引号脱同步攻击；引号内换行符。）

命令替换阻断是最关键的防御。COMMAND_SUBSTITUTION_PATTERNS 数组包含 12 种模式，覆盖了所有已知的命令替换形式。

（代码从略：这段代码定义了 12 种命令替换的检测模式，包括两种进程替换写法、Zsh 的等号展开、美元符加圆括号的命令替换、美元符加花括号的参数替换、美元符加方括号的旧式算术展开、Zsh 风格的参数展开、Zsh 风格的 glob 限定符、带命令执行的 glob 限定符、Zsh 的 always 块，以及作为纵深防御的 PowerShell 注释语法。）

这些模式里有几个要单独说明。第一个是 Zsh 等号展开 =curl：在 Zsh 中，=cmd 会展开为 cmd 的完整路径，等同于 which cmd 的结果。攻击者可以用 =curl evil.com 绕过 Bash(curl:*) 的 deny 规则，因为权限系统解析到的基命令是 =curl 而不是 curl。第二个是 Zsh glob 限定符，即 (e: 和 (+ 开头的写法：Zsh 的 glob 限定符可以在文件名匹配过程中执行任意代码，这是一个常被忽略的代码执行向量。第三个是 PowerShell 注释语法 <#：虽然 Claude Code 不在 PowerShell 中执行命令，但作为纵深防御，以防未来引入 PowerShell 执行路径。

为了避免误报，例如引号内文本本身包含命令替换写法时被错误检测，安全验证器使用 extractQuotedContent 函数先剥离引号内的内容。这个函数逐字符迭代，跟踪单引号和双引号状态，产出三种变体。withDoubleQuotes 仅剥离单引号内容。fullyUnquoted 剥离所有引号内容。unquotedKeepQuoteChars 剥离内容但保留引号字符本身，用于检测引号邻接。

Zsh 危险命令同样有专门的防御。ZSH_DANGEROUS_COMMANDS 集合包含 18 个命令，其中几类值得说明。第一个是 zmodload，即 Zsh 模块加载器，它是多种攻击的入口：zsh/mapfile 借数组赋值做不可见的文件 I/O；zsh/zpty 提供伪终端命令执行；zsh/net/tcp 可用 ztcp 外泄网络数据；zsh/files 的内置 rm、mv、ln、chmod 绕过二进制检查。第二个是 emulate 加 -c 参数，一个等效于 eval 的结构，可以执行任意代码。第三类是 sysopen、sysread、syswrite、sysseek，来自 zsh/system 模块的细粒度文件描述符操作。第四类是 zf_ 开头的内置命令，即 zf_rm、zf_mv、zf_ln、zf_chmod 等，是 zsh/files 模块提供的内置文件操作，绕过了对外部二进制的权限检查。

Tree-sitter AST 解析提供了结构化的命令分析能力。当 tree-sitter-bash 可用时，parseCommandRaw 将命令解析为 AST，parseForSecurityFromAst 从中提取 SimpleCommand 数组，也就是已解析引号的简单命令列表。如果 AST 分析发现命令结构过于复杂，包含命令替换、展开、复杂控制流，它返回 too-complex，触发 ask 行为，即要求用户确认。当 tree-sitter 不可用时，比如某些平台，系统回退到基于正则的遗留解析路径。checkSemantics 函数在 AST 级别校验语义，识别语法合法但语义危险的命令。

4.6.2 多层权限系统（bashPermissions.ts）

bashToolHasPermission 位于 src/tools/BashTool/bashPermissions.ts，是整个代码库中最复杂的权限函数，实现了以下分层处理。

第 1 步，AST 解析与复杂度判断。首先尝试使用 tree-sitter 解析命令。解析结果分为三种。第一种是 simple，命令结构简单，可以按子命令逐一检查。第二种是 too-complex，包含复杂结构，比如嵌套替换、管道链等，无法保证安全，跳过详细分析，直接进入 ask 路径。第三种是 parse-unavailable，tree-sitter 不可用，回退到遗留的 splitCommand_DEPRECATED 正则拆分。

第 2 步，子命令拆分与上限保护。复合命令，例如用两个和号、两个竖线连接的多段命令，会被拆分为子命令数组，每个子命令独立检查。为防止 CPU 耗尽，恶意构造的复合命令可能导致正则拆分产生指数级增长的子命令，拆分数量设有硬性上限，常量 MAX_SUBCOMMANDS_FOR_SECURITY_CHECK 的值为 50。超过 50 个子命令时，系统放弃逐一分析，直接返回 ask。这是一个安全的回退：无法证明安全的就让用户决定。

第 3 步，安全环境变量剥离。在匹配权限规则之前，系统会先剥离命令前面的安全环境变量赋值。例如 NODE_ENV=prod npm run build 会剥掉 NODE_ENV=prod，剩下 npm run build 用于规则匹配。SAFE_ENV_VARS 集合包含 32 个已知安全的变量：Go 系列如 GOEXPERIMENT、GOOS、GOARCH，Rust 系列如 RUST_BACKTRACE、RUST_LOG，Node 的 NODE_ENV，以及 LANG、LC_ALL 等 locale 变量。内部构建另有 ANT_ONLY_SAFE_ENV_VARS 扩展集，在 USER_TYPE 为 ant 时生效。

为什么要区分安全和不安全的环境变量？因为 MY_VAR=val command 中的 MY_VAR 可能影响命令行为，比如 LD_PRELOAD=evil.so curl，不能无条件剥离。但 NODE_ENV=prod 是无害的，如果不剥离，用户设置的 Bash(npm run:*) 规则就无法匹配到 NODE_ENV=prod npm run build。

第 4 步，前缀提取与规则建议。getSimpleCommandPrefix 从命令中提取稳定的 2 词前缀，用于可复用的权限规则。举几个例子。第一个例子，命令 git commit -m 加提交说明，提取的前缀是 git commit，建议的规则是 Bash(git commit:*)。第二个例子，命令 npm run build，提取的前缀是 npm run，建议的规则是 Bash(npm run:*)。第三个例子，命令 NODE_ENV=prod npm run build 在剥离安全环境变量后，提取的前缀同样是 npm run，建议的规则还是 Bash(npm run:*)。第四个例子，命令 ls -la，提取的前缀是 null，因为 -la 是标志不是子命令，所以仅提供精确匹配规则。第五个例子，形如 bash -c 加任意命令的调用，提取的前缀是 null，因为 bash 被阻止生成前缀规则，所以不建议前缀规则。

注意最后一个例子：bash、sh、sudo、env 等裸 shell 前缀被显式阻止生成前缀规则，因为 Bash(bash:*) 等同于 Bash(*)，这会意外允许所有命令。

第 5 步，复合命令权限聚合。对于复合命令，所有子命令必须独立通过权限检查。任一子命令被 deny 则整体 deny，任一子命令需要 ask 则整体 ask。建议规则的数量有上限，常量 MAX_SUGGESTED_RULES_FOR_COMPOUND 的值为 5。超过 5 条时，权限弹窗退化为 similar commands 描述，而非列出每一条。这避免了用户在一条串联命令链中保存 10 多条规则的混乱体验。

4.6.3 沙箱模式（shouldUseSandbox.ts）

BashTool 支持在沙箱中执行命令，限制文件系统访问、网络和进程能力。沙箱决策逻辑 shouldUseSandbox 在以下条件下返回 false，即不使用沙箱。第一，SandboxManager 的 isSandboxingEnabled 返回 false，也就是全局禁用。第二，命令设置了 dangerouslyDisableSandbox 为 true 且策略允许绕过。第三，命令匹配用户配置的 excludedCommands 排除列表。

平台支持方面：macOS 使用 sandbox-exec 配置文件，Linux 使用 bubblewrap，简称 bwrap，提供类似 landlock 的限制。排除命令的处理涉及复合命令拆分、环境变量剥离和通配符匹配，与权限系统共享相同的命令解析基础设施。

4.6.4 sed 验证（sedValidation.ts）

sed 命令有专门的验证层，防止它被用作绕过 FileEditTool 权限的后门。验证采用白名单策略，只有已知安全的模式才会自动放行。

第一种安全模式是纯行打印，必须有 -n 标志。例如打印第 5 行；打印第 1 到 10 行；或者打印多个指定行。

第二种安全模式是替换表达式。例如把 foo 替换为 bar 并全局生效，但标志仅允许 g、p、i、I、m、M 和数字 1 到 9。

被阻断的危险操作包括六类。第一类是 w 和 W 标志，用于文件写入。第二类是 e 和 E 标志，用于命令执行，sed 可以通过 e 标志执行 shell 命令。第三类是感叹号，用于地址取反。第四类是花括号块，即 sed 脚本块。第五类是非 ASCII 字符，用于 Unicode 同形字检测。第六类是反斜杠分隔符，存在潜在的解析混淆。

当 sed 命令包含文件参数时，例如对某个文本文件执行就地替换，就地编辑标志 -i 必须明确存在，且需要文件写入权限。纯读模式下的 sed 不允许操作文件参数。

4.6.5 路径验证与破坏性命令警告

路径验证由 pathValidation.ts 实现，它从涉及文件路径的命令中提取路径参数，验证它们是否在允许的工作目录范围内。覆盖 cd、rm、mv、cp、cat、grep 等约 36 类命令，也就是 PATH_EXTRACTORS 的键。不同命令有不同的路径提取规则：cd 将所有参数拼接为一个路径；find 收集第一个非全局标志之前的路径；grep 和 rg 在解析完 pattern 参数后收集文件路径。所有命令都尊重 POSIX 的 -- 分隔符，在它之后的所有参数都是位置参数而非标志。

对于危险的删除路径，比如 rm -rf / 或 rm -rf ~ 这类强制递归删除，无论用户有什么已保存的规则，都始终需要显式批准。这是一个不可覆盖的安全硬限制。

破坏性命令警告由 destructiveCommandWarning.ts 实现，它是一个纯信息层，不影响权限决策，只在权限弹窗中显示额外警告。检测的模式共有九类，每类对应一种命令模式和一条警告消息。第一类针对 Git 数据丢失，模式是 git reset --hard，警告可能丢弃未提交的更改。第二类针对 Git 历史覆写，模式是 git push --force 或其简写，警告可能覆写远程历史。第三类针对 Git 安全旁路，模式是 --no-verify，警告可能跳过安全钩子。第四类针对 Git 提交覆写，模式是 git commit --amend，警告可能改写最后一次提交。第五类是递归强制删除，模式是 rm -rf，警告可能递归强制删除文件。第六类和第七类针对数据库，前者匹配 DROP TABLE 或 TRUNCATE，警告可能删除或截断数据库对象；后者匹配不带 WHERE 子句的 DELETE FROM 语句，警告可能删除所有行。第八类和第九类针对基础设施，前者匹配 kubectl delete，警告可能删除 Kubernetes 资源；后者匹配 terraform destroy，警告可能摧毁 Terraform 基础设施。

4.6.6 后台任务管理

BashTool 支持两种后台执行模式。

第一种是显式后台化。模型在参数中设置 run_in_background 为 true，命令从一开始就作为 LocalShellTask 异步执行。输出流式写入任务输出文件，模型可通过 TaskGetTool 轮询结果。

第二种是自动后台化。在长时间运行对话的助手模式下，阻塞命令在 15 秒后自动转为后台执行，对应常量 ASSISTANT_BLOCKING_BUDGET_MS 的值为 15000。系统调用 backgroundExistingForegroundTask 将前台任务移至后台，释放主循环继续处理。这防止了一个长时间运行的 npm install 或 make build 阻塞整个对话。

4.6.7 命令语义（commandSemantics.ts）

BashTool 不只是机械地检查退出码，它理解不同命令的语义约定。标准 Unix 约定中，退出码 0 表示成功，非 0 表示失败，但这并不普适。

按退出码 0、1 和 2 及以上三种情况来看。第一个是 grep 和 rg：退出码 0 表示找到匹配，退出码 1 表示无匹配而不是错误，退出码 2 及以上才是真正的错误。第二个是 diff：退出码 0 表示无差异，退出码 1 表示有差异而不是错误，退出码 2 及以上才是真正的错误。第三个是 test 和它的中括号等价形式：退出码 0 表示条件为真，退出码 1 表示条件为假而不是错误，退出码 2 及以上是语法错误。第四个是 find：退出码 0 表示成功，退出码 1 是部分成功，某些目录不可访问，退出码 2 及以上才是真正的错误。

interpretCommandResult 根据命令名查表解读退出码，避免模型将 grep 的无匹配误判为执行失败而发起不必要的重试。

4.6.8 命令分类（UI 展示）

BashTool 还维护了一个命令分类系统，用于 UI 中的折叠显示。isSearchOrReadBashCommand 函数分析管道中的每个部分，只有当所有部分都是搜索或读取命令时才标记为可折叠。

分类有四类。第一类是搜索命令，包括 find、grep、rg、ag、ack、locate、which、whereis，UI 上折叠为 Searched。第二类是读取命令，包括 cat、head、tail、less、more、wc、stat、file、jq、awk、cut、sort、uniq、tr，UI 上折叠为 Read。第三类是列表命令，包括 ls、tree、du，UI 上折叠为 Listed。第四类是语义中性命令，包括 echo、printf、true、false 和冒号，在分类中被跳过，不影响管道分类。

语义中性命令在管道分类中直接跳过。例如 ls dir 之后接一条 echo 再接 ls dir2 的命令序列，仍然算读取操作，不会因为 echo 而变成不可折叠。

4.7 AgentTool 深度解析

AgentTool 负责派生子 Agent，是多 Agent 架构的核心。

（代码从略：这是 AgentTool 的输入 Schema，包含七个字段。description 是 3 到 5 词的任务描述；prompt 是子 Agent 的任务指令；subagent_type 指定专用 Agent 类型；model 可选 sonnet、opus 或 haiku；run_in_background 表示异步执行；name 是可寻址的队友名称；isolation 可选 worktree 或 remote 两种隔离模式。）

子 Agent 生命周期

子 Agent 从创建到执行经历 6 个阶段，每个阶段都有精确的决策逻辑。

首先，进行 Agent 定义查找，按 subagent_type 匹配，并做 MCP 需求过滤和权限过滤。然后进行模型解析，优先级是参数指定高于定义默认、定义默认高于继承父级。接着搭建隔离环境，worktree 模式通过 git worktree 创建，remote 模式做 CCR 部署检查；随后组装工具池，由内置工具加 MCP 工具按 Agent 定义过滤而成。最后渲染系统提示词，把 Agent 定义模板与工作目录、操作系统、Shell、Git 状态等环境信息合并，然后执行并返回。返回方式有四种：同步模式下结果直接嵌入父对话；异步模式下通过 LocalAgentTask 文件轮询；队友模式下通过 Tmux 或 iTerm2 会话工作；远程模式下通过 CCR 的 WebSocket 通信。

第一阶段是 Agent 定义查找。如果提供了 subagent_type，系统按类型名匹配预定义的 Agent 定义，比如 coder、researcher。匹配时还会检查 Agent 定义所需的 MCP 服务是否可用、当前权限模式是否允许。未匹配到定义时使用通用默认配置。

第二阶段是模型解析。模型选择遵循三级优先级链：调用参数中的 model 字段优先级最高，没有就用 Agent 定义中的默认模型，再没有才继承父 Agent 当前使用的模型。这种设计允许开销敏感的任务使用 haiku，复杂任务升级到 opus。

第三阶段是隔离环境搭建。worktree 模式通过 git worktree add 创建独立工作树，子 Agent 在隔离的文件系统视图中工作，避免与父 Agent 的文件编辑冲突。remote 模式检查 CCR，也就是 Claude Code Remote 的环境可用性，准备远程部署。

第四阶段是工具池组装。子 Agent 的工具池不一定与父 Agent 相同。Agent 定义可以指定工具白名单或黑名单，例如 researcher 类型可能只获得只读工具。MCP 工具也按 Agent 定义的需求过滤。

第五阶段是系统提示词渲染。将 Agent 定义中的提示词模板与环境信息合并。注入内容包括当前工作目录、操作系统类型、Shell 类型、Git 仓库状态等。这确保子 Agent 对执行环境有准确的认知。

第六阶段是执行与返回。根据调用方式分为四种模式。同步模式下，子 Agent 在当前进程内直接执行，结果嵌入父对话的 tool_result 中。异步模式下，创建 LocalAgentTask，子 Agent 将结果写入临时文件，父 Agent 通过 TaskGetTool 轮询获取。队友模式下，通过 Tmux 或 iTerm2 创建新的终端会话，子 Agent 作为独立进程并行工作，可通过 SendMessageTool 通信。远程模式下，通过 CCR 创建远程执行环境，返回 WebSocket URL，结果异步回传。

4.8 大结果处理机制

当工具输出超过 maxResultSizeChars 时，Claude Code 不会将全部内容注入对话上下文。第一步，将完整结果保存到 ~/.claude/projects 下按项目目录和会话 ID 划分的 tool-results 文件夹，按项目、按会话隔离。第二步，模型接收文件路径预览加截断指示符。第三步，模型可通过 FileReadTool 按需读取完整内容。

这避免了上下文膨胀，同时保持完整数据的可达性。

各工具的典型阈值

不同工具的 maxResultSizeChars 根据其输出特征设定。第一个是 BashTool，阈值 30000 字符，因为 Shell 命令输出可能非常大，比如全盘 find。第二个是 GrepTool，阈值 20000 字符，因为大范围搜索可能匹配数千行。第三个是 WebFetchTool，阈值 100000 字符，因为网页内容长度不可控。第四个是 FileReadTool，阈值为无限大，它的输出已由 maxTokens 分页自限，显式豁免落盘，因为落盘后再让模型读回来就成了循环。

需要注意：工具声明的 maxResultSizeChars 还会被 getPersistenceThreshold 用 DEFAULT_MAX_RESULT_SIZE_CHARS 等于 50000 做取小钳制。也就是说声明值超过 50000 也不会真正生效，这也是下一节侧栏所说的 50000 上限。

MCP 工具的大结果处理

MCP 工具的输出有额外处理。当输出超过 25000 token 时，默认值来自 DEFAULT_MAX_MCP_OUTPUT_TOKENS，可用 MAX_MCP_OUTPUT_TOKENS 环境变量覆盖，truncateMcpContent 会就地截断并追加一条截断提示，说明输出已超出 token 上限。其中的图片块会先由 compressImageBlock 压缩，尽量塞进剩余的 token 预算，不会整体丢弃，相关代码在 utils/mcpValidation.ts。至于超大的文本结果，则和其他工具一样走通用的 tool-results 落盘机制，见 4.8 节；并不存在专门的二进制 blob 落盘后返回路径的通道。注意这里是 25000 token 而非 25KB。

整体设计遵循按需读取模式，英文是 on-demand read pattern。模型先看到结果的摘要和位置信息，只在需要详细数据时才通过 FileReadTool 主动拉取。这将一次性的上下文爆炸转化为可控的增量读取。

设计决策：工具结果的三级大小限制。

源码中定义了三个递进的限制层，位于 src/constants/toolLimits.ts。第一级是 DEFAULT_MAX_RESULT_SIZE_CHARS，值为 50000，是单个工具结果的默认上限，超过则持久化到磁盘。第二级是 MAX_TOOL_RESULT_TOKENS，值为 100000，约合 400KB，是任何工具都不能超过的绝对上限。第三级是 MAX_TOOL_RESULTS_PER_MESSAGE_CHARS，值为 200000，是单条消息中所有工具结果的聚合上限。

为什么需要三级？因为并发工具执行时，5 个工具各返回 50K 字符的结果就是 250K，超过单消息限制。聚合上限确保即使多个工具并发返回大结果，注入到对话中的总数据量也不会让上下文窗口失控。

4.9 MCP 工具集成

MCP，全称 Model Context Protocol，它的工具通过桥接层无缝集成到 Claude Code 的工具系统中。

桥接工具

Claude Code 提供了四个桥接入口。第一个是 MCPTool，用于调用单个 MCP 工具。第二个是 ListMcpResourcesTool，用于列出 MCP 资源。第三个是 ReadMcpResourceTool，用于读取 MCP 资源内容。第四个是 createMcpAuthTool，用于 OAuth 认证处理。

传输机制与服务端配置类型

传输层由 TransportSchema 定义，共 6 种传输机制，代码位于 services/mcp/types.ts。

（代码从略：TransportSchema 是一个枚举，包含 6 种传输。stdio 是标准输入输出，用于子进程 MCP 服务端；sse 是 Server-Sent Events，基于 HTTP 流式；sse-ide 是面向 IDE 扩展的 SSE 变体；http 是 HTTP 传输；ws 是双向实时的 WebSocket；sdk 是进程内的 SDK 原生传输。）

而服务端配置 McpServerConfigSchema 是一个包含 8 种配置的可辨识联合类型。在上面 6 种传输之外，还多了面向 IDE 的 ws-ide，对应 McpWebSocketIDEServerConfig，以及面向 Claude.ai 代理的 claudeai-proxy。换句话说：传输枚举 6 种、服务端配置类型 8 种，二者不是同一份清单。

连接状态机

MCP 连接的状态流转如下。初始状态是 Pending。连接成功就进入 Connected，连接失败就进入 Failed。Connected 状态下如果 OAuth 需要，会进入 NeedsAuth；认证完成后回到 Connected。此外，用户也可以直接把服务置为 Disabled 状态。

客户端实例有 memoize 缓存，避免重复初始化。HTTP 404 加 JSON-RPC -32001 用于检测会话过期。

OAuth 支持

MCP 集成支持三阶段 OAuth。第一阶段是标准 OAuth 2.0 加 PKCE，自动 Token 轮换，30 秒超时。第二阶段是跨应用访问，简称 XAA，基于 OIDC，实现企业 IdP 集成，一次登录多个 MCP 服务端。第三阶段是 Token 验证，主动刷新接近过期的 Token，并用 macOS Keychain 缓存。

配置与作用域

（代码从略：这段 JSON 展示了 mcpServers 配置的写法。本地服务端用 command 和 args 指定启动命令，例如用 node 运行 my-mcp-server.js；远程服务端用 url 字段指定服务地址。）

MCP 服务端配置支持 7 种作用域，分别是 local、user、project、dynamic、enterprise、claudeai 和 managed。

MCP 工具在 assembleToolPool 阶段与内置工具合并，去重后统一注册。输出超过默认的 25000 token 时就地截断并附截断提示，图片块会先尝试压缩进剩余预算，代码在 utils/mcpValidation.ts；超大文本结果与其他工具一样走 tool-results 落盘机制。

设计决策：为什么 MCP 适合 Agent 生态？

MCP 的核心设计思想是协议而非 SDK。任何语言、任何进程都可以实现 MCP 服务端，只要遵循 JSON-RPC 协议。这与 Claude Code 的工具系统天然互补：内置工具是深度集成，直接访问进程内状态；MCP 工具是广度扩展，连接外部能力。6 种传输机制、8 种服务端配置类型，对应的是现实世界的多样性。本地工具用 stdio，零网络开销；远程服务用 HTTP 或 WebSocket，支持认证和断线重连；IDE 插件用 SSE-IDE，适配 VS Code 的进程模型。配置的 7 层作用域从本地到企业，覆盖了不同组织规模的管理需求。

4.10 工具搜索与延迟加载

并非所有 60 多个工具都会在每次 API 调用时进入模型的上下文窗口。ToolSearchTool 与 API 侧的 defer_loading，也就是延迟加载，配合工作。Claude Code 工具定义里的 shouldDefer 字段对应序列化到请求中的 defer_loading 标记。这里要澄清一个常见误解：defer_loading 决定的是什么进入上下文，而不是请求里发送什么。

具体来说有五点。第一，请求的 tools 数组始终包含所有工具的完整定义，包括延迟加载的那些，因为 API 需要它们在服务端执行工具搜索，并在模型发现工具时把 tool_reference 块展开成完整定义。第二，defer_loading 为 false 时，也就是默认情况，工具立即进入模型上下文。第三，defer_loading 为 true 时，工具只有在模型通过 ToolSearch 发现之后才进入上下文。第四，工具的 searchHint 字段提供搜索提示。第五，regex 和 bm25 两种搜索变体都会匹配工具名称、描述、参数名和参数描述。

延迟加载的收益在于：减少进入上下文从而进入系统提示词前缀的工具数量，压缩每一轮实际处理的 token，同时让前缀保持稳定以提升 prompt cache 命中率。

searchHint 字段

每个可延迟加载的工具都可以定义 searchHint 字符串，用于提高工具被发现的概率。例如，一个 Jupyter Notebook 编辑工具可能把 searchHint 设为 notebook jupyter ipynb cell。当模型调用 ToolSearch 时，搜索算法同时匹配工具名称、描述、参数名、参数描述以及 searchHint。

ToolSearchTool 查询语法

ToolSearchTool 支持三种查询模式。第一种是精确选择，写法是 select 冒号加工具名列表，按名称直接加载指定工具，例如 select:Read,Edit,Grep。第二种是关键词搜索，直接输入关键词，返回最匹配的若干个工具，例如 notebook jupyter。第三种是名称前缀约束，用加号加前缀开头，要求工具名包含前缀，再按关键词排序，例如 +slack send。

select 模式最常用，当模型已经知道需要哪个工具时直接按名称加载，零搜索开销。关键词模式适用于探索性场景，比如需要一个处理数据库的工具。

对提示词缓存的影响

defer_loading 决定的是什么进入上下文窗口，而不是请求里发送什么，这正是它缓存收益的来源。既然每次请求的 tools 数组始终包含全部工具，包括延迟的，工具集本身并不构成缓存变动因素；真正参与 prompt cache 缓存键的是系统提示词前缀。

API 在服务端把延迟工具排除在系统提示词前缀之外，整个机制分三步。第一步，对话开始时，前缀只包含非延迟工具的完整定义，形成一段稳定的缓存前缀。第二步，模型发现延迟工具时，API 在对话历史中就地追加一个 tool_reference 块，并在传给模型前把它展开为完整工具定义。第三步，前缀从未被改动，因此 prompt cache 全程命中，动态加载工具不会破坏缓存。

这意味着你可以用一小撮始终加载的工具开启对话并保持缓存命中，让模型按需发现更多工具，而每一轮都能维持同样的缓存命中。

与 strict mode 的配合：严格工具调用的语法约束是从完整工具集构建的，与哪些工具被延迟无关。所以 defer_loading 与 strict mode 组合使用时不需要重新编译语法，也就是 grammar，prompt cache 与 grammar cache 都能在动态加载工具时被保留。

使用建议有三条，来自官方文档。第一，绝不要给工具搜索工具本身设置 defer_loading 为 true，否则模型将无法通过搜索发现任何其他工具。第二，保持 3 到 5 个最常用的工具为非延迟，让模型无需先搜索即可直接调用。第三，由于搜索覆盖工具名称、描述、参数名和参数描述，写 searchHint 与工具描述时应尽量覆盖这些维度。

兼容性提示：并非所有模型厂商都支持延迟加载。defer_loading 与工具搜索是 API 服务端实现的能力，Anthropic API 原生支持，但第三方网关、本地模型、其他厂商的兼容层未必实现这一机制。如果所使用的提供商不支持该功能，建议关闭延迟加载，将所有工具设为非延迟，避免不必要的副作用。

4.11 设计洞察

第一个洞察：统一接口的力量。所有工具，无论是直接访问进程内状态的内置工具、通过 JSON-RPC 连接的 MCP 工具，还是运行在独立 VM 中的 REPL 工具，都共享同一个 Tool 泛型接口。这意味着从输入验证、权限检查、Hook 到执行和结果处理，整条流水线对所有工具完全相同。新增一个 MCP 服务端不需要修改任何执行逻辑，只需要实现 call 并提供 inputSchema。这种统一性是 Claude Code 能够在不增加系统复杂度的前提下从 20 个工具扩展到 60 多个工具的关键。

第二个洞察：安全语义编码为类型。isReadOnly、isDestructive、isConcurrencySafe 不只是布尔标记，它们是参与运行时决策的活跃方法。isReadOnly 接收工具输入作为参数，这意味着同一工具对不同输入可以有不同的安全语义。例如，BashTool 对 ls 返回 isReadOnly 为 true，对 rm 返回 false。这种细粒度的输入感知安全标记让并发调度器和权限系统能做出更精确的决策，而非对整个工具一刀切。

第三个洞察：渲染即工具。每个工具自带 React 渲染方法，比如 renderToolUseMessage 和 renderToolResultMessage，而非由统一的渲染器根据工具类型分发。这种自描述渲染设计意味着工具最了解自己的输入输出应该如何展示。FileEditTool 渲染带颜色的 diff，BashTool 渲染带退出码的终端输出，GrepTool 渲染带行号的搜索结果。新增工具时只需实现自己的 UI.tsx，不需要修改任何全局渲染逻辑。

第四个洞察：Feature Gate 的编译时裁剪。通过 Bun 的 feature 编译时宏和死代码消除，外部构建从物理上不包含内部工具的代码。运行时的 isInternal 检查可以绕过；编译产物中干脆不存在相关代码，即使拿到构建产物也无法还原。

第五个洞察：Fail-closed 安全默认值。TOOL_DEFAULTS 中 isConcurrencySafe 默认 false 和 isReadOnly 默认 false 的选择源于失败模式的不对称性。如果一个工具实际上是只读的但被标记为非只读，也就是忘记 opt-in，后果是用户收到不必要的权限弹窗，烦人但安全。反过来，如果一个有写入副作用的工具被错误标记为只读，也就是忘记 opt-out，后果是它可能在没有权限检查的情况下与其他写入工具并发执行，导致数据损坏，危险且隐蔽。这种不对称性决定了默认值必须选择安全但可能过度限制的方向。

第六个洞察：纵深防御的分层验证。BashTool 的安全不依赖任何单一防线。它有 7 层以上重叠的安全机制：Tree-sitter AST 解析、正则模式匹配、引用内容提取、路径约束验证、sed 白名单验证、沙箱隔离、权限规则系统。每一层都有已知的局限性，正则可以被精心构造的输入绕过，tree-sitter 可能不可用，沙箱可能不被平台支持，但设计哲学是：任何单层都可以失败，攻击者却需要同时绕过所有层才能成功。这使得利用难度呈指数级增长。

第七个洞察：Prompt Cache 稳定性作为架构约束。多个看似无关的设计决策实际上都被同一个隐形约束驱动，就是 prompt cache 命中率。相关决策包括：assembleToolPool 的分区排序，防止 MCP 工具变动污染内置工具的缓存键；backfillObservableInput 只修改 UI 层的浅拷贝而非 API 输入，防止修改消息内容导致缓存失效；ToolSearch 延迟加载，延迟工具不进入系统提示词前缀，前缀保持稳定，缓存持续命中。

缓存未命中意味着 API 需要重新处理数千个 token 的系统提示词，增加延迟和成本。这个经济学约束深刻地塑造了架构，但不阅读多个组件很难察觉到。

4.12 工具 UI 渲染模式

每个工具不仅定义执行逻辑，还自带完整的 UI 渲染能力。这是渲染即工具设计理念的体现，工具最了解自己的输入输出应该如何展示。

渲染方法一览

每个工具可以定义 4 到 6 个 React 渲染方法。第一个是 renderToolUseMessage，用于渲染工具调用过程，在模型发出 tool_use 时触发。第二个是 renderToolResultMessage，用于渲染工具执行结果，在工具完成执行后触发。第三个是 renderToolUseRejectedMessage，用于渲染权限拒绝信息，在用户拒绝权限请求时触发。第四个是 renderToolUseErrorMessage，用于渲染错误信息，在工具执行出错时触发。第五个是 renderGroupedToolUse，用于合并渲染多个同类调用，在同类型工具连续调用时触发。

renderGroupedToolUse：批量合并渲染

当模型连续调用多个相同类型的工具，比如连续 5 个 FileReadTool，逐个渲染会占据大量终端空间。renderGroupedToolUse 方法将这些调用合并为一个紧凑的视图。

（代码从略：这是合并渲染后的界面输出示例，只占一行，内容大意是读取了 5 个文件，后面列出 src/agent.ts、src/tools.ts、src/cli.ts 等文件名。）

（代码从略：这是逐个渲染时的输出示例，共 5 行，每行各报告一个文件被读取，分别是 src/agent.ts、src/tools.ts、src/cli.ts、src/prompt.ts 和 src/session.ts。）

这不仅节省屏幕空间，也让用户更容易理解模型的意图：它在批量阅读文件，而非在一个一个读文件。

backfillObservableInput：输入回填

工具输入在到达 UI 之前会经过 backfillObservableInput 处理。这个方法在不修改发送给 API 的实际输入的前提下，为 UI 观察者扩展输入信息。

典型例子：模型调用 FileEditTool 时只传入相对路径 src/agent.ts，但 UI 展示时需要完整路径 /home/user/project/src/agent.ts。backfillObservableInput 将当前工作目录补全到路径中供 UI 使用，但 API 侧的 tool_use block 保持原样。

为什么不直接修改 API 输入？因为 prompt cache 稳定性。API 请求中的消息内容是缓存键的一部分，任何修改都会导致缓存失效。backfillObservableInput 只影响 UI 展示层，API 层的消息原封不动。

具体示例：FileEditTool 的 UI 渲染

FileEditTool 的 UI.tsx 根据操作类型渲染不同的视觉效果。创建文件时，标题显示 Create，渲染完整文件内容并附带语法高亮。编辑文件时，标题显示 Update，渲染结构化的 diff 补丁，新增行标绿、删除行标红。出错时显示匹配失败的上下文，帮助用户理解为什么 old_string 未能在文件中找到匹配。

每个工具的 UI.tsx 都遵循同样的模式：导入工具的类型定义，实现相应的 render 系列方法，返回 React 节点。这种规范化结构让新工具的 UI 开发有模板可循。

动手实践：在 claude-code-from-scratch 项目的 src/tools.ts 中，约 325 行代码实现了 7 个核心工具。对比本章的 60 多个工具体系，这是理解最小可用工具集的最佳起点。参见教程第 2 章：工具系统。

上一章：上下文工程。下一章：技能系统。
