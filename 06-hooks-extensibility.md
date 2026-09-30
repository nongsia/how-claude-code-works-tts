---
title: 第 7 章：Hooks 与可扩展性（朗读版）
---

第 7 章：Hooks 与可扩展性

本文是 docs/06-hooks-extensibility.md 的朗读版，表格与代码已转为口语描述，内容未增删。

Hooks 是 Claude Code 的事件驱动扩展机制——在不修改源码的前提下，注入自定义逻辑到关键生命周期节点。

想象一下这些场景：每次 Claude 执行 git push 之前自动运行 lint 检查；每次编辑文件后在后台跑测试，只在测试失败时中断 Claude；或者把所有工具调用发送到公司审计系统。这些都是 Hooks 的典型用法。

Hooks 的核心设计理念是：Agent Loop 的每个关键节点都暴露一个事件，外部代码可以监听这些事件并注入行为。这和 Git Hooks（pre-commit、post-merge）、Webpack Plugins 的设计理念一脉相承，但 Claude Code 面对的问题更复杂——它需要处理权限控制、异步长任务、多 Agent 协调等场景，因此 Hook 系统的设计远比传统的"前后拦截器"复杂得多。

本章主要内容如下。第一，7.1 事件全景，介绍 27 种 Hook 事件的分类与触发时机。第二，7.2 Hook 类型，介绍 4 种可配置 Hook，即 Command、Prompt、Agent、HTTP，以及 Callback 和 Function 这 2 种编程式 Hook。第三，7.3 Matcher 匹配器，介绍三级匹配机制与 if 条件的配合。第四，7.4 执行引擎，介绍 6 阶段流水线，包括信任检查、匹配、去重、并行执行、输出解析、结果聚合。第五，7.5 到 7.9 的高级主题，包括 JSON 输出协议、信任模型与安全、PermissionRequest 深度解析、Stop Hook 和实战模式。

7.1 Hook 事件全景

为什么是这 27 种事件？

Claude Code 的 Hook 事件设计遵循一个原则：覆盖 Agent Loop 完整生命周期的所有关键决策点。回顾第 2 章的 Agent Loop，一次完整的交互要顺着这样一条链走：用户先输入，模型据此推理，然后调用工具；工具这一步内部又分权限检查、执行、返回结果；模型拿到结果再决定是否继续，最后给出输出。每个环节都可能需要外部干预，因此每个环节都要有对应的 Hook 事件。

源码中定义了完整的事件列表，位置在 src/entrypoints/sdk/coreTypes.ts。（代码从略：这段代码导出了名为 HOOK_EVENTS 的常量，列出全部 27 种事件。前 13 种是 PreToolUse、PostToolUse、PostToolUseFailure、Notification、UserPromptSubmit、SessionStart、SessionEnd、Stop、StopFailure、SubagentStart、SubagentStop、PreCompact、PostCompact；后 14 种是 PermissionRequest、PermissionDenied、Setup、TeammateIdle、TaskCreated、TaskCompleted、Elicitation、ElicitationResult、ConfigChange、WorktreeCreate、WorktreeRemove、InstructionsLoaded、CwdChanged、FileChanged。）

按功能分类。第一类是工具生命周期。PreToolUse 在工具执行前触发，PostToolUse 在工具执行成功后触发，PostToolUseFailure 在工具执行失败后触发，这三者的 Matcher 匹配值都是工具名，比如 Write 或 Bash。第二类是权限系统。PermissionRequest 在权限判定时触发，PermissionDenied 在自动分类器拒绝时触发，匹配值也都是工具名。第三类是通知。Notification 在系统通知触发时触发，匹配值是通知类型。第四类是会话生命周期。SessionStart 在会话开始时触发，匹配值是触发源，即 startup、resume、clear 或 compact。SessionEnd 在会话结束时触发，匹配值是原因。UserPromptSubmit 在用户提交输入时触发，没有匹配值。第五类是模型响应。Stop 在模型决定停止时触发，没有匹配值。StopFailure 在 API 调用失败时触发，匹配值是 error。第六类是 Agent 协调。SubagentStart 在子 Agent 启动时触发，SubagentStop 在子 Agent 停止时触发，匹配值都是 Agent 类型。TeammateIdle 在协作 Agent 空闲时触发，没有匹配值。第七类是任务系统。TaskCreated 在任务创建时触发，TaskCompleted 在任务完成时触发，都没有匹配值。第八类是压缩。PreCompact 在上下文压缩前触发，PostCompact 在上下文压缩后触发，匹配值是触发方式，即 manual 或 auto。第九类是 MCP 交互。Elicitation 对应 MCP 用户询问，ElicitationResult 对应询问结果，匹配值都是 MCP 服务器名。第十类是环境变化。ConfigChange 在配置文件变更时触发，匹配值是来源。CwdChanged 在工作目录变更时触发，没有匹配值。FileChanged 在被监听文件变更时触发，匹配值是文件名，也就是路径的 basename。InstructionsLoaded 在指令文件加载时触发，匹配值是加载原因。第十一类是工作区。Setup 对应仓库初始化或维护，匹配值是触发方式，即 init 或 maintenance。WorktreeCreate 在 Worktree 创建时触发，WorktreeRemove 在 Worktree 移除时触发，这两者都没有匹配值。

这里的 Matcher 匹配值要专门说一下——它告诉你，当你在配置中把 matcher 写成 Write 时，系统实际拿什么值来比较。对于工具相关事件，matcher 匹配的是工具名；对于 SessionStart，匹配的是触发源；对于 Notification，匹配的是通知类型。这个映射关系在 getMatchingHooks 的一个 switch 语句中定义。

为什么需要这么多事件？

初看 27 种事件可能觉得过多，但每个事件都有明确的使用场景。第一，工具前后事件，即 PreToolUse 和 PostToolUse，这是最核心的扩展点，前置 Hook 可以阻止执行、修改输入，后置 Hook 可以执行检查、注入上下文。第二，会话事件，即 SessionStart 和 SessionEnd，用于初始化环境、清理资源、上报审计日志。第三，环境变化事件，包括 FileChanged、CwdChanged、ConfigChange，用于响应外部变化，实现"文件保存后自动 lint"等工作流。第四，Agent 协调事件，包括 SubagentStart、SubagentStop、TeammateIdle，用于在多 Agent 场景中注入协调逻辑。

7.2 Hook 类型

Claude Code 支持四种可配置的 Hook 类型和两种编程式 Hook 类型。前四种可以写在 settings.json 中，后两种仅在 SDK 和插件内部使用。

六种类型的总览如下。第一，Command 类型，持久化在 settings.json 中，执行方式是 spawn 一个 Shell 子进程、通过标准输入和标准输出通信，适用于日志、lint、CI 触发等绝大多数场景。第二，Prompt 类型，持久化在 settings.json 中，执行方式是单轮 LLM 调用，返回 ok 或 not-ok，适用于需要语义理解的安全检查或代码审查。第三，Agent 类型，持久化在 settings.json 中，执行方式是多轮 Agent Loop，可以调用工具做验证，适用于复杂验证流程，比如运行测试、类型检查。第四，HTTP 类型，持久化在 settings.json 中，执行方式是向外部端点发送 POST 请求，适用于 Webhook 通知、审计日志、企业合规。第五，Callback 类型，仅存在于内存中，由 SDK 或插件注册，执行方式是在进程内直接调用异步函数，适用于内部埋点、文件跟踪、commit 归因。第六，Function 类型，仅存在于内存中、按会话级注册，执行方式是在进程内调用并按 sessionId 隔离，适用于 Agent Hook 的结构化输出强制。

在讲具体类型之前，先看一个最简单的 Hook 配置示例，对整体格式有个直观认识。（代码从略：这个示例写在用户主目录的 settings.json 里。hooks 对象的键是事件名，此处为 PreToolUse，值是一个数组；数组元素的 matcher 字段设为 Bash 做可选的匹配过滤，hooks 字段列出该匹配下要执行的 Hook，这里是一条 command 类型的 Hook，命令只输出一句"即将运行 Bash 命令"的提示。）

这个结构不复杂：hooks 对象的键是事件名，比如 PreToolUse；值是数组，每个元素含两个字段——matcher 做可选的匹配过滤，hooks 列出该匹配下要执行的 Hook。

命令 Hook（Command）

这是最常用的类型。执行一条 Shell 命令，通过 stdin 接收 JSON 输入，通过 stdout 返回 JSON 结果，通过退出码表达成功、失败或阻塞。

（代码从略：Command Hook 的配置字段包括 type 设为 command、要执行的 Shell 命令 command、用权限规则语法做二次过滤的可选 if 字段、指定 bash 或 powershell 的可选 shell 字段且默认 bash、以秒计的超时 timeout、执行时的 spinner 提示 statusMessage、执行一次后自动移除的 once、异步执行不阻塞的 async，以及异步执行且退出码为 2 时唤醒模型的 asyncRewake。）

工作原理，即 execCommandHook 的实现，分五步。第一步，进程创建：调用 spawn 创建子进程。Shell 的选择逻辑是：如果指定了 shell 为 powershell，就用 pwsh 并加上 -NoProfile 和 -NonInteractive 参数。否则走 spawn 并开启 shell 选项，Unix 上即 /bin/sh，Windows 上换成 Git Bash，由 findGitBashPath 找到。注意它并不读取用户的 $SHELL，schema 描述里的 "$SHELL" 说法与实际 spawn 实现不符。第二步，输入传递：将 Hook 的结构化输入序列化为 JSON，通过 stdin 传入子进程。这份输入包含 session_id、tool_name、tool_input 等字段，因此 Hook 脚本读一下 stdin 就能拿到完整的上下文。第三步，环境变量：子进程继承当前环境变量。如果是插件 Hook，额外注入两个变量——CLAUDE_PLUGIN_ROOT 指向插件根目录，CLAUDE_PLUGIN_DATA 指向插件数据目录；命令里的 ${CLAUDE_PLUGIN_ROOT} 占位符也会被替换。第四步，输出收集：等待进程退出，收集 stdout 和 stderr。第五步，结果解析：根据退出码和 stdout 内容决定 Hook 结果，详见 7.4 节。

适用场景：日志记录、文件同步、CI/CD 触发、shell 脚本集成、自定义 linter。

提示词 Hook（Prompt）

调用 LLM 从语义层面做判断。适用于需要"理解"而非简单模式匹配的场景。

（代码从略：Prompt Hook 的配置字段包括 type 设为 prompt、提示词 prompt，其中 $ARGUMENTS 占位符会被替换为 JSON 输入、权限规则语法的 if 过滤、指定模型的 model 字段，默认使用小快模型，比如 Haiku、以秒计的超时 timeout，默认 30 秒，以及 statusMessage 和 once。）

工作原理，即 execPromptHook 的实现，分四步。第一步，将 $ARGUMENTS 占位符替换为 Hook 输入的 JSON 字符串。第二步，构建消息数组，可选地带上对话历史，再调用 queryModelWithoutStreaming 做单轮、无流式的请求。第三步，系统提示词要求模型返回 ok 为 true，或者 ok 为 false 加原因的 JSON。第四步，解析模型返回，ok 为 false 时映射为阻塞错误。

一个关键设计细节：Prompt Hook 直接调用 createUserMessage 而不经过 processUserInput——因为后者会触发 UserPromptSubmit Hook，导致无限递归。

适用场景：语义安全检查，比如"这个 SQL 查询是否可能删除数据？"；代码审查，比如"这个修改是否符合项目规范？"。

Agent Hook

与 Prompt Hook 类似，但以多轮 Agent 模式运行——它可以调用工具来验证条件，不仅仅是"想一想"。

（代码从略：Agent Hook 的配置字段包括 type 设为 agent、验证指令 prompt，同样支持 $ARGUMENTS 占位符、if 过滤、指定模型的 model 字段，默认使用 Haiku、以秒计的超时 timeout，默认 60 秒，以及 statusMessage 和 once。）

它与 Prompt Hook 的关键区别可以这样对比。第一，调用方式：Prompt Hook 通过 queryModelWithoutStreaming 做单轮调用，Agent Hook 通过 query 走多轮 Agent Loop。第二，能否调用工具：Prompt Hook 不能，只有 LLM 推理；Agent Hook 能，可以读文件、运行命令来验证。第三，默认超时：Prompt Hook 是 30 秒，Agent Hook 是 60 秒。第四，输出格式：Prompt Hook 强制返回包含 ok 和 reason 的 JSON；Agent Hook 通过注册结构化输出工具，返回包含 ok 和 reason 的结果。

Agent Hook 使用 registerStructuredOutputEnforcement 注册一个函数 Hook，确保 Agent 在结束时必须调用结构化输出工具返回结果。这是一个"Hook 嵌套 Hook"的设计——Agent Hook 本身在执行过程中注册临时的 Function Hook 来约束 Agent 行为。

适用场景：复杂验证流程——例如"运行测试并确认全部通过"、"检查编辑的文件是否能通过类型检查"。

HTTP Hook

向外部服务发送 POST 请求，适合与企业基础设施集成。

（代码从略：HTTP Hook 的配置字段包括 type 设为 http、POST 端点 url、if 过滤、以秒计的超时 timeout，默认 10 分钟、支持 $VAR 环境变量插值的 headers、允许插值的环境变量白名单 allowedEnvVars，以及 statusMessage 和 once。）

工作原理，即 execHttpHook 的实现，分六步。第一步，URL 白名单检查：如果配置了 allowedHttpHookUrls 策略，先检查 URL 是否匹配允许的模式，不匹配直接拒绝，不发任何请求。第二步，Header 环境变量插值：遍历 headers，匹配 $VAR_NAME 或 ${VAR_NAME} 模式。只有在 allowedEnvVars 中列出的变量才会被替换，其他变量替换为空字符串。这防止了项目级 settings.json 中的恶意 Hook 窃取 $HOME、$AWS_SECRET_ACCESS_KEY 等敏感变量。第三步，CRLF 注入防护：插值后的 header 值会被去除回车、换行和空字符，防止恶意环境变量注入额外的 HTTP 头。第四步，代理支持：自动检测 sandbox 代理和环境变量代理，即 HTTP_PROXY 和 HTTPS_PROXY，通过代理发送请求。第五步，SSRF 防护：不通过代理时，使用 ssrfGuardedLookup 防止请求发往内网地址。第六步，响应解析：HTTP Hook 必须返回 JSON，这跟 Command Hook 不同——后者可以返回纯文本。空 body 被视为空对象，即成功且无特殊指令。

有一个重要限制：HTTP Hook 不支持 SessionStart 和 Setup 事件。原因是在 headless 模式下，这两个事件触发时 sandbox 的 structuredInput 消费者尚未启动，HTTP 请求会死锁。

适用场景：Webhook 通知、审计日志上报、第三方审批系统、合规检查。

回调 Hook（Callback），仅限 SDK 和插件

编程式函数，在进程内直接执行，不经过 spawn、HTTP 等 I/O 操作。

（代码从略：Callback Hook 的配置包括 type 设为 callback、一个异步回调函数 callback，它接收输入、工具调用 ID、中断信号、序号和上下文，返回 Hook 的 JSON 输出，另有以秒计的超时 timeout，以及标记为内部 Hook 的 internal 字段，internal 为 true 时启用快速路径优化。）

Callback Hook 为什么这么快？Claude Code 在 executeHooks 中有一个针对内部 Hook 的快速路径优化。（代码从略：这段代码来自 src/utils/hooks.ts。isInternalHook 判断一个 Hook 是否是 internal 为 true 的 callback 类型。代码先把匹配结果中所有非内部 Hook 过滤出来，如果一个都没剩，说明这一批全是内部 callback，就进入快速路径：在一个循环里直接逐个调用它们的回调函数然后返回，跳过 JSON 序列化、AbortSignal 创建、进度事件和结果处理，不经过常规的输出处理流程。）

这个优化将内部 Hook 的开销从约 6 微秒降低到约 1.8 微秒，下降了 70%。快速路径的触发条件是：所有匹配 Hook 都是 internal 为 true 的 callback，比如文件访问跟踪、commit 归因这类内置埋点。它们在每次工具调用时都触发，累积起来差距很大；而普通的、非 internal 的 SDK callback 和 function Hook 只要出现一个，整批就会走常规流程。

函数 Hook（Function），仅限会话内

类似 Callback，但作用域限定在特定会话内，防止跨 Agent 泄漏。

（代码从略：Function Hook 的配置包括 type 设为 function、可选的 id、一个接收消息列表和中断信号、返回布尔值或其 Promise 的回调 callback、callback 返回 false 时显示的错误消息 errorMessage，以及超时和 spinner 提示字段。）

主要用途是 Agent Hook 的结构化输出强制，确保 Agent 必须通过特定工具返回结果。通过 addFunctionHook 注册，removeFunctionHook 移除，按 sessionId 隔离——这确保验证 Agent 的函数 Hook 不会泄漏到主 Agent。

通用字段说明

有几个字段在多种 Hook 类型中出现，值得单独解释。

if 条件是一个比 matcher 更精细的过滤器。Matcher 匹配工具名，比如 Bash；if 则用权限规则语法匹配工具的具体输入，比如 Bash(git *) 只在 Bash 工具执行 git 命令时触发。if 条件在 prepareIfConditionMatcher 中解析：它调用工具的 preparePermissionMatcher 对工具输入做模式匹配，复用了权限系统的匹配引擎。要注意 if 只能在工具相关事件上求值，也就是 PreToolUse、PostToolUse、PostToolUseFailure、PermissionRequest。在其他事件上，带 if 条件的 Hook 会被 ifFilteredHooks 过滤器直接剔除、永不执行——源码里 ifMatcher 为 undefined 时就直接返回 false，而不是"忽略 if、让 Hook 照常触发"。

once 字段：如果为 true，Hook 执行一次后自动从配置中移除。适用于一次性的初始化或验证。

statusMessage 字段：Hook 执行时在 spinner 中显示的自定义消息。默认显示命令内容，但对于复杂命令或包含敏感信息的命令，自定义消息更友好。

7.3 Matcher 匹配器

Matcher 是 Hook 系统的路由机制——决定一个 Hook 是否应该响应某个事件。

配置格式

（代码从略：HookMatcher 类型包含两个字段，可选的匹配模式 matcher，不设置则匹配所有，以及匹配时要执行的 Hook 列表 hooks。）

配置示例。（代码从略：这段配置在 PreToolUse 事件下放了两个匹配项。第一个 matcher 为 Bash，命中时输出一句"使用了 Bash 工具"的提示；第二个 matcher 是用竖线分隔的 Write 和 Edit，命中时输出一句"文件被修改"的提示。）

三种匹配模式

matchesPattern 函数实现了三级匹配，按顺序尝试。（代码从略：这段代码来自 src/utils/hooks.ts。matcher 为空或为星号时直接匹配所有；如果 matcher 只含字母、数字、下划线和竖线，就视为简单模式——带竖线的按竖线分割后逐个精确匹配，不带竖线的直接做全等比较；否则把它当作正则表达式来测试。）

三种模式的设计体现了渐进复杂度。第一，精确匹配，例如只写 Write，这是最常用的方式，匹配单个工具。第二，管道分隔，例如把 Write、Edit、Read 用竖线连起来，可以匹配多个工具，语义是"或"。第三，正则表达式，例如匹配以 Bash 开头的任意工具名，或者用分组精确匹配 Write 或 Edit，适用于复杂模式匹配。

为什么不直接全部用正则？因为绝大多数用户只需要精确匹配。正则的字符串模式检测确保简单的工具名不会被意外当成正则解析——检测条件是字符串只含字母、数字、下划线和竖线——比如 Bash 这个词不会触发正则引擎。

Matcher 与 if 条件的配合

Matcher 和 if 构成了两层过滤。这段结构图描述了过滤流程：事件触发后，先做 Matcher 过滤，粗粒度地匹配工具名或事件类型，不匹配就直接跳过，不会 spawn 进程；通过后再做 if 条件过滤，细粒度地匹配工具的具体输入参数，不匹配同样跳过、不 spawn 进程；两关都过了，才真正执行 Hook。

举例。（代码从略：这段配置的 matcher 是 Bash，Hook 命令输出"检测到 git 命令"的提示，同时 if 条件设为 Bash(git push*)，即只在 Bash 工具执行 git push 时触发。）

这个配置的匹配过程分四步。第一步，PreToolUse 事件触发，工具名是 Bash，matcher 匹配通过。第二步，检查 if 条件，也就是 Bash(git push*)，解析权限规则后检查工具输入的命令是否匹配 git push 加星号的模式。第三步，如果用户执行的是 git push origin main，匹配通过，执行 Hook。第四步，如果用户执行的是 git status，匹配失败，跳过。

这里的性能关键在于：两层过滤都在 spawn 子进程之前完成。如果一个 PreToolUse 事件触发了 10 个 Hook 配置，但只有 2 个通过了 matcher 加 if 的双重过滤，系统只会 spawn 2 个进程。这是"零成本抽象"——不触发的 Hook 完全没有运行时开销。

7.4 Hook 执行引擎

关键文件：src/utils/hooks.ts，这是核心调度。

Hook 执行经过 6 个阶段，下面逐一展开每个阶段的实现细节。整个流程是：首先，Hook 事件触发后经过一个快速存在性检查，即 hasHookForEvent，没有任何配置就直接返回，有配置才继续。然后依次经过第一阶段的信任检查 shouldSkipHookDueToTrust、第二阶段的 Matcher 与 if 匹配 getMatchingHooks、第三阶段按 hookDedupKey 去重、第四阶段的输入构建与并行执行。最后是第五阶段的输出解析与退出码语义，以及第六阶段的结果聚合与事件发射。

Stage 0：快速存在性检查

在进入完整的 Hook 流程之前，hasHookForEvent 提供了一个轻量级的短路判断。（代码从略：它依次检查三个来源——快照里该事件的配置、已注册的 SDK 或插件 Hook、当前会话的会话 Hook，任何一处存在且非空就返回 true，否则返回 false。）

这个检查故意做成过度近似：它不检查 matcher 是否匹配，不检查 managedOnly 策略——只要有任何配置存在就返回 true。假阳性只是多走一步完整匹配路径；假阴性则会跳过应执行的 Hook，所以宁可多查不可漏查。

这个优化的价值在于：绝大多数事件根本没有配置任何 Hook。一个没有配置 FileChanged Hook 的项目，每次文件变化事件都能在几微秒内短路返回，省掉了 createBaseHookInput 的路径拼接和 getMatchingHooks 的配置遍历。

Stage 1：信任检查

shouldSkipHookDueToTrust 是安全底线——所有 Hook 都需要工作区信任。（代码从略：它先判断是否交互式会话，SDK 等非交互模式下信任隐式成立，直接返回 false；交互模式下检查用户是否接受过信任对话框，没有接受就返回 true，表示跳过 Hook。）

为什么要卡得这么死？Hooks 从 .claude/settings.json 读取配置并执行任意命令。如果不检查信任，恶意仓库可以通过在 .claude/settings.json 中注入 Hook 来执行代码——用户只要 clone 并打开仓库，Hook 就会自动运行。

这个设计是被历史漏洞逼出来的。第一是 SessionEnd Hook 泄露：用户 clone 一个恶意仓库，打开 Claude Code，看到信任对话框后点了拒绝，随即退出。但 SessionEnd Hook 在退出时执行，并不检查信任，于是恶意 Hook 照样跑了一遍。第二是 SubagentStop Hook 提前执行：子 Agent 在信任对话框弹出前就跑完了，SubagentStop 事件随之触发，Hook 就落在了未经信任的工作区里。

修复方案简单而有效：在 executeHooks 的最开头统一检查信任，所有 Hook 无一例外，都必须在工作区信任建立后才能执行。源码注释直接点明了这一点：This centralized check prevents RCE vulnerabilities for all current and future hooks。

Stage 2：Matcher 匹配与 Hook 收集

getMatchingHooks 是整个引擎中逻辑最复杂的函数，它需要做五件事。第一，收集所有来源的 Hook 配置：快照配置、注册的 SDK 或插件 Hook、会话 Hook、函数 Hook。第二，根据事件类型确定 matchQuery：通过 switch 语句从 hookInput 中提取匹配值。第三，Matcher 匹配过滤：对每个 HookMatcher，检查 matcher 是否匹配 matchQuery。第四，if 条件过滤：使用 prepareIfConditionMatcher 生成匹配闭包，逐个检查。第五，特殊限制：HTTP Hook 在 SessionStart 和 Setup 事件中被过滤掉。

matchQuery 的提取逻辑，即源码 getMatchingHooks 中的 switch 语句。（代码从略：PreToolUse、PostToolUse、PostToolUseFailure、PermissionRequest、PermissionDenied 这五种事件取工具名；SessionStart 取触发源，即 startup、resume、clear、compact 之一；Setup 取触发方式，即 init 或 maintenance；Notification 取通知类型；SubagentStart 和 SubagentStop 取 Agent 类型；FileChanged 取文件路径的 basename，也就是不含路径的文件名。）

注意 FileChanged 用的是 basename——只匹配文件名，不匹配路径。这意味着 matcher 设为 .env 时，会匹配任何目录下的 .env 文件。

Stage 3：Hook 去重

当 Hook 配置在多个来源中重复出现时（比如用户设置和项目设置都定义了同一条 Hook），去重机制确保不会重复执行。去重键由 hookDedupKey 生成：取插件根目录或技能根目录作为前缀，两者都没有时用空字符串，后面拼上一个空字符分隔符，再拼上 Hook 的载荷内容。

去重的核心设计有三点。第一，同源 Hook 去重：来自 settings 的 Hook 没有 pluginRoot 和 skillRoot，共享空字符串前缀，相同命令只保留最后合并的那个。第二，跨源 Hook 不去重：插件 A 和插件 B 可能都有指向 hook.sh 的同一条命令模板，展开后指向不同文件；去重键包含 pluginRoot，确保它们不会被错误地合并。第三，不同 if 条件不去重：即使命令相同，if 条件不同也是不同的 Hook。

Last-wins 语义：new Map 在键冲突时保留最后一个条目。对于 settings Hook，这意味着后合并的配置（如项目设置）覆盖先合并的用户设置。

Callback 和 Function Hook 跳过去重——每个回调函数都是唯一的，去重没有意义。

Stage 4：输入构建与并行执行

先说输入构建。createBaseHookInput 构建所有 Hook 共用的基础输入。（代码从略：基础输入包含会话 ID session_id、对话记录文件路径 transcript_path、当前工作目录 cwd，以及可选的权限模式 permission_mode、子 Agent ID agent_id 和 Agent 类型 agent_type。）

agent_type 有一个值得注意的优先级逻辑：子 Agent 的类型（来自 toolUseContext）优先于主线程的 --agent 标志。这样 Hook 可以通过 agent_id 是否存在来区分"主 Agent 的工具调用"和"子 Agent 的工具调用"。

JSON 输入采用惰性序列化：Hook 输入只序列化一次，通过闭包共享给同一批次的所有 Hook。（代码从略：getJsonInput 用一个缓存变量保存序列化结果，第一次调用时把 hookInput 用 jsonStringify 转成字符串并缓存，之后直接复用；序列化抛错时同样把错误缓存下来。）

如果一个事件触发了 5 个 Command Hook，hookInput 只被 jsonStringify 执行一次。

再看并行执行。所有匹配的 Hook 通过 hookPromises 的 map 异步生成器并行启动，用 all 等待所有结果。这意味着 5 个 Hook 的执行时间取决于最慢的那个，而不是 5 个的总和。每个 Hook 有独立的超时控制，createCombinedAbortSignal 会把父级 signal 和 Hook 自己的超时合到一起。

执行分三种模式。

同步模式是默认的：等待进程退出，收集 stdout 和 stderr，解析输出。虽然多个 Hook 之间是并行的，但每个 Hook 自身是同步等待结果的。

异步模式，即 async 设为 true：Hook 进程在后台运行，通过 registerPendingAsyncHook 注册到全局的 AsyncHookRegistry，立即返回 success。Agent Loop 在每轮循环中调用 checkForAsyncHookResponses 轮询已完成的异步 Hook，将结果注入对话。这里的超时取值要分清两条路径。配置里写 async 为 true 的 Hook，后台超时沿用该 Hook 的 timeout 字段，也就是 hook.timeout 乘以 1000，缺省则是 TOOL_HOOK_EXECUTION_TIMEOUT_MS，等于 10 分钟，转入后台时把这个值作为 asyncTimeout 传入。只有当 Hook 通过 stdout 自行返回 async 为 true、却没带 asyncTimeout 时，才落到 AsyncHookRegistry 里缺省 15000 的这条 15 秒兜底上。

异步唤醒模式，即 asyncRewake 设为 true，最特殊，专为"后台检查加按需中断"场景设计。（代码从略：这段代码来自 executeInBackground。asyncRewake 的 Hook 绕过 AsyncHookRegistry，后台命令结束后检查退出码；如果退出码为 2，表示阻塞性错误，就把 Hook 名和错误输出包进 system-reminder，通过 enqueuePendingNotification 以任务通知的方式入队，等待唤醒模型。）

工作流程分四步。第一步，Hook 进程在后台运行，不阻塞当前操作。第二步，退出码 0 表示静默成功，不打扰模型。第三步，退出码 2 表示阻塞性错误，通过 enqueuePendingNotification 注入 system-reminder 唤醒模型。第四步，模型在下一轮看到这个错误后可以做出响应，比如修复失败的测试。

为什么用退出码 2 而不是 1？这是一个精心设计的约定：0 表示成功，1 表示一般错误，用户可见但不打扰模型，2 表示需要模型关注的阻塞性错误。这让长时间运行的检查，比如 CI 构建、集成测试，只在真正发现问题时才中断模型的工作流。

还有一个实现细节值得一提：asyncRewake Hook 故意不调用 shellCommand 的 background 方法——因为 background 会触发 taskOutput 的 spillToDisk，将输出写入磁盘文件。但后续需要通过 getStdout 和 getStderr 读取内存中的输出来构建通知消息，spillToDisk 会把 stderr 从内存中清除，让它返回空字符串。

Stage 5：输出解析与退出码语义

Hook 的输出协议由两部分组成：退出码和 stdout 内容。

退出码语义

对于非 JSON 输出的 Command Hook，退出码是唯一的通信渠道。退出码的含义分四档。第一，退出码 0 表示成功，stdout 显示在 transcript 中，一般不影响模型；SessionStart 和 UserPromptSubmit 两个事件例外，下面马上讲。第二，退出码 1 表示一般错误，stderr 显示给用户，但不传递给模型。第三，退出码 2 表示阻塞性错误，stderr 显示给用户，并且传递给模型，会阻止操作。第四，其他退出码都按一般错误处理，等同退出码 1，stderr 显示给用户、不传递给模型。

退出码 0 有一个例外：在 SessionStart 和 UserPromptSubmit 两个事件下，退出码 0 且 stdout 非空时，Claude Code 会把 stdout 包成 system-reminder 注入对话、对模型可见，消息形如"某个 Hook 名加 hook success 加内容"，走的是 utils/messages.ts 的 hook_success 分支。这也是"不写 JSON、纯 echo 就能给模型注入上下文"的惯用法。其余事件下退出码 0 的 stdout 只进 transcript、不进模型上下文。

退出码 1 和 2 的区别值得强调：退出码 1 只是"告诉用户出了点问题"，属于 non-blocking，模型不知道也不关心；退出码 2 是"告诉模型这里有问题，必须处理"，属于 blocking，会阻止当前工具的执行或模型的停止。

这个区分很实用。Linter 警告用退出码 1，用户看得到，但工作流不被打断；安全检查失败用退出码 2，模型必须知道并处理。

stdout 解析

parseHookOutput 会分情况解析 stdout。（代码从略：它先把 stdout 去掉首尾空白；如果不以左花括号开头，就当作纯文本返回，不尝试 JSON 解析；否则尝试 JSON 解析加 Zod schema 验证，验证通过就返回解析结果，验证失败则返回纯文本并附带 validationError，其中包含期望的 schema；解析抛出异常时同样按纯文本处理。）

设计要点有三个。第一，以左花括号开头才尝试 JSON 解析——简单的 echo done 不会被误解析。第二，JSON 解析后还要经过 Zod schema 验证——确保字段名和类型都正确。第三，验证失败时会给 schema 提示：当 JSON 格式不对时，错误信息中直接展示期望的完整 schema。这对 DX，也就是开发者体验，很友好——Hook 作者不需要查文档就能看到正确的格式应该是什么样的。

HTTP Hook 用 parseHttpHookOutput 解析，规则略有不同：空 body 被视为空对象，表示成功；非空内容必须是有效 JSON。

Stage 6：结果聚合与事件发射

多个并行 Hook 的结果通过 AggregatedHookResult 合并。（代码从略：这个类型汇总各 Hook 的结果，字段包括可选的阻塞错误 blockingError、是否阻止继续的 preventContinuation、停止原因 stopReason、取值为 allow、deny、ask 或 passthrough 的 permissionBehavior、可来自多个 Hook 的额外上下文列表 additionalContexts、修改后的工具输入 updatedInput，以及修改后的 MCP 输出 updatedMCPToolOutput。）

聚合走的是 for await 循环配合 all 的流式路径——每个 Hook 一有结果就立即产出，不必等所有 Hook 跑完。这意味着一个 Hook 的阻塞错误可以立即传播，而不必等待其他更慢的 Hook。

事件发射用于 UI 更新和可观测性。第一，emitHookStarted 在 Hook 开始执行时发射，用于 spinner 显示。第二，emitHookResponse 在 Hook 完成时发射，包含 stdout、stderr、退出码和结局。第三，startHookProgressInterval 让长时间运行的 Hook 定期发射进度更新，用于在远程模式下推送实时输出。

超时配置

超时配置的默认值如下。第一，一般 Hook 默认 10 分钟，来自 TOOL_HOOK_EXECUTION_TIMEOUT_MS。第二，SessionEnd Hook 默认只有 1.5 秒，来自 SESSION_END_HOOK_TIMEOUT_MS_DEFAULT。第三，Prompt Hook 默认 30 秒，硬编码。第四，Agent Hook 默认 60 秒，硬编码。第五，HTTP Hook 默认 10 分钟，来自 DEFAULT_HTTP_HOOK_TIMEOUT_MS。第六，配置里写 async 为 true 的 Hook，后台超时沿用该 Hook 的 timeout 字段，缺省 10 分钟，转入后台时以 asyncTimeout 传入。第七，Hook 通过 stdout 自声明异步、又没带 asyncTimeout 的，走 AsyncHookRegistry 缺省 15000 对应的 15 秒兜底。第八，自定义超时可以在 Hook 定义里配置 timeout 字段，单位是秒。

SessionEnd 超时极短（1.5 秒），因为用户正在退出，不应被 Hook 阻塞。可通过环境变量 CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS 覆盖。

另外要澄清一点：asyncTimeout 属于 Hook 的 JSON 输出协议字段，也就是 Hook 在 stdout 里声明的，并不是 settings.json 的可配置项——settings 里的异步相关字段只有 timeout、async 和 asyncRewake。


7.5 Hook JSON Output Schema

Hook 通过 stdout 输出 JSON 来控制 Claude Code 的行为。完整 schema 定义在 src/types/hooks.ts 中，使用 Zod 验证。

通用字段

（代码从略：通用输出字段分三组。流程控制组：continue 设为 false 会阻止 Claude 继续，对应 preventContinuation；suppressOutput 设为 true 时隐藏 stdout 输出，不记入 transcript；stopReason 是 continue 为 false 时的停止原因消息。决策字段组，向后兼容：decision 取 approve 表示允许，取 block 表示阻止并报错；reason 是决策原因。反馈组：systemMessage 是显示给用户的警告消息。）

事件特定输出，即 hookSpecificOutput

hookSpecificOutput 是一个 discriminated union，通过 hookEventName 字段区分不同事件的输出格式。

PreToolUse 的输出最丰富，可以控制权限、修改输入、注入上下文。（代码从略：它的 hookSpecificOutput 以 PreToolUse 为事件名，字段包括权限决策 permissionDecision，取值 allow、deny 或 ask，决策原因 permissionDecisionReason，修改工具输入的 updatedInput，以及附加上下文 additionalContext。）

PermissionRequest 的输出结构与 PreToolUse 不同，使用嵌套的 decision 对象。（代码从略：decision 有两种形态。允许时 behavior 为 allow，可以带 updatedInput 修改输入、updatedPermissions 注入新权限规则；拒绝时 behavior 为 deny，可以带 message 说明拒绝原因、interrupt 标志中断操作。）

PostToolUse 的输出可以注入上下文或替换 MCP 工具输出。（代码从略：字段包括 additionalContext，以及用于替换 MCP 工具原始输出的 updatedMCPToolOutput。）

SessionStart 的输出可以设置初始消息和文件监听。（代码从略：字段包括 additionalContext、自动注入的初始用户消息 initialUserMessage，以及注册 FileChanged 监听路径的 watchPaths。）

UserPromptSubmit、Setup、SubagentStart、PostToolUseFailure、Notification 这五种事件的输出格式相同，只有一个可选的 additionalContext 字段。

PermissionDenied 的输出只有一个可选的 retry 字段，用于触发重试。

Elicitation 和 ElicitationResult 对应 MCP 交互响应。（代码从略：字段包括 action，取值 accept、decline 或 cancel，以及内容对象 content。）

CwdChanged 在工作目录变更时，可以注册新的文件监听路径。（代码从略：字段是 watchPaths，注册 FileChanged 监听的绝对路径。）

FileChanged 在被监听文件变更时，同样可以更新监听路径。（代码从略：字段也是 watchPaths，更新 FileChanged 监听的绝对路径。）

WorktreeCreate 在 Worktree 创建时，可以指定工作树路径。（代码从略：字段是 worktreePath，即新创建的工作树路径。）

异步响应方面，Hook 也可以在输出中声明 async 为 true，并可选地带上 asyncTimeout，表示结果稍后到达。

常用字段组合速查

常用字段组合速查如下。第一，要批准工具执行，输出 hookSpecificOutput，事件名为 PreToolUse，permissionDecision 设为 allow。第二，要拒绝工具执行，同样的事件名下把 permissionDecision 设为 deny，并用 permissionDecisionReason 说明原因，比如"不安全"。第三，要修改工具输入，在 allow 的基础上再加 updatedInput，比如把 command 改成 git push --dry-run。第四，要阻止继续，输出 continue 设为 false，并用 stopReason 说明原因，比如"检测到安全问题"。第五，要注入上下文，hookSpecificOutput 的事件名用 UserPromptSubmit，additionalContext 写入要注入的内容，比如"当前 linter 有 3 个警告"。

数据流跟踪示例

以一个 PreToolUse Hook 拒绝 rm -rf 命令为例，跟踪完整的数据流。首先，模型调用 Bash 工具，命令是 rm -rf /tmp/data，Agent Loop 触发 PreToolUse 事件，executePreToolHooks 构建 hookInput，里面包含事件名 PreToolUse、工具名 Bash、工具输入中的命令、会话 ID 和当前工作目录等字段。然后 getMatchingHooks 查找匹配的 Hook：matcher 为 Bash，匹配工具名通过；if 条件为 Bash(rm *)，经 preparePermissionMatcher 对这条命令做匹配，同样通过。接着 execCommandHook 执行 Hook 命令：spawn 子进程，Unix 走 /bin/sh，Windows 走 Git Bash，把 JSON 输入写进 stdin，然后等待退出。Hook 脚本在 stdout 返回一段 JSON，声明事件名为 PreToolUse、权限决策为 deny，原因是 rm -rf 命令被安全策略禁止；parseHookOutput 看到它以左花括号开头，进行 JSON 解析，Zod 验证通过。最后 processHookJSONOutput 处理这段 JSON：确认事件名是 PreToolUse，权限决策是 deny，于是结果的 permissionBehavior 定为 deny，blockingError 指向"rm -rf 命令被安全策略禁止"这条消息。工具执行被阻止，模型收到的错误消息是 PreToolUse 冒号 Bash hook error，后面跟着这条安全策略原因。

7.6 信任模型与安全

Hook 配置快照

captureHooksConfigSnapshot 在启动时冻结 Hook 配置。这意味着三点。第一，Hook 定义在整个会话期间不变。第二，即使 .claude/settings.json 在会话中被修改，比如被恶意代码修改，Hook 也不会动态更新。第三，配置变更时 updateHooksConfigSnapshot 会重新捕获，但只在受控的场景中触发。

三层来源与策略控制

getHooksFromAllowedSources 从三个来源合并 Hook 配置。合并前先过托管策略这一关，也就是企业管理员配置的 policySettings。如果策略把 disableAllHooks 设为 true，所有 Hook 一律禁用，包括管理员自己配置的。如果把 allowManagedHooksOnly 设为 true，就只使用托管策略中的 Hook，忽略用户、项目、插件的 Hook。默认情况下合并所有来源，最终配置由托管策略 Hook、用户主目录下 settings.json 里的 Hook、以及项目 .claude/settings.json 里的 Hook 合并而成。

三种策略模式的设计对应不同的企业安全需求。第一，disableAllHooks，效果是禁用一切 Hook，包括托管策略自己的，适用于安全锁定环境、完全不信任 Hook 机制的场景。第二，allowManagedHooksOnly，效果是只运行管理员在 policySettings 中定义的 Hook，适用于企业合规环境，防止用户或仓库注入不受控的 Hook。第三，默认模式，合并所有来源，适用于灵活的开发环境。

allowManagedHooksOnly 的影响范围很广——它不仅挡住用户设置和项目设置里的 Hook，还挡住插件 Hook，也就是 pluginRoot 存在时直接跳过，以及 Agent、Skill frontmatter 里定义的那些会话 Hook。但它不挡 SDK 注册的 callback Hook，那些是运行时内部机制，不算用户配置。

7.7 PermissionRequest Hook 深度解析

这是最强大的 Hook 类型——可以程序化地控制工具权限，在权限系统章节中参与竞速机制。

输入

（代码从略：输入包含事件名 PermissionRequest、工具名称 tool_name、工具输入参数 tool_input、会话 ID session_id、当前工作目录 cwd 和权限模式 permission_mode。）

输出

PermissionRequest 的 hookSpecificOutput 使用嵌套的 decision 对象，与 PreToolUse 的 permissionDecision 字段不同。（代码从略：decision 有两种形态。允许时 behavior 为 allow，可以同时用 updatedInput 修改输入、用 updatedPermissions 注入权限规则；拒绝时 behavior 为 deny，可以附带拒绝消息 message 和中断标志 interrupt。）

四种能力

PermissionRequest Hook 远不止简单的 allow 和 deny，它有四种能力。第一，审批决策：behavior 取 allow 或 deny。第二，输入修改：通过 updatedInput 修改工具的输入参数，比如强制添加 --dry-run 标志。第三，规则注入：通过 updatedPermissions 动态持久化新的权限规则，不只是本次生效，而是在整个会话中持续生效。第四，操作中断：interrupt 为 true 时立即中断当前操作。

与权限系统的竞速

PermissionRequest Hook 参与权限系统的竞速机制（详见第 12 章权限与安全）——与 UI 确认对话框和 LLM 分类器同时运行，先完成的获胜。

时序图展示了这场竞速。首先，工具调用把输入参数发给 PermissionRequest Hook，同时向 UI 发出确认显示、向 LLM 分类器发出分类请求，三路并行竞速。然后，Hook 最先完成，返回 allow 并修改输入，而此时用户还没点确认框、分类器还在计算。最后，一个 ResolveOnce 守卫保证只有一个结果生效，Hook 先完成，Hook 的决定就生效。


7.8 Stop Hook：采样后验证

Stop Hook 在模型决定停止循环时触发，也就是模型返回纯文本、不再发起工具调用时，它可以阻止终止并强制继续。流程是这样的：模型返回纯文本、没有工具调用时，Stop Hook 触发；如果 Hook 返回 allow 或没有阻塞错误，就正常终止；如果返回 deny 或以退出码 2 结束，系统会把 blockingError 注入对话，状态转移为 stop_hook_blocking，Agent Loop 继续运行。

这使得自动化工作流可以实现"做完了才能停"的语义。例如：（代码从略：在 Stop 事件下配置一条 command Hook，命令先用静默方式运行 npm test，如果测试失败，就向 stderr 输出 Tests failed 并以退出码 2 退出。）

每次模型准备停止时都自动跑一遍测试。测试通过就允许停止；测试失败则以退出码 2 退出，模型收到 Tests failed 消息，被迫继续修复。

7.9 实战模式

模式 1：CI 构建检查（asyncRewake）

场景：每次编辑文件后自动运行测试，但不阻塞编辑工作流，只在测试失败时提醒模型。

为什么选 asyncRewake？测试可能需要几十秒。如果用同步 Hook，模型每编辑一个文件就要等几十秒。asyncRewake 让测试在后台运行，模型继续编辑其他文件，只在测试失败时才被中断。

（代码从略：在 PostToolUse 事件下用 matcher 为 Edit 匹配编辑操作，配置一条 asyncRewake 为 true 的 command Hook，命令运行 npm test，退出码非零就以 2 退出，否则以 0 退出。）

工作流程是这样：每次文件编辑后，后台自动运行测试。测试通过（退出码 0）就静默。测试失败（退出码 2）则唤醒模型并注入错误信息，模型自动修复。

模式 2：上下文注入（additionalContext）

场景：每次用户提交输入时，自动运行 linter 并将结果注入为额外上下文。

为什么选 UserPromptSubmit 加 additionalContext？这让模型在开始工作前就知道现有的 lint 问题，可以在修改时顺手修复。

（代码从略：在 UserPromptSubmit 事件下配置一条 command Hook，命令运行 npx eslint，以 compact 格式检查 src 目录并把输出存进变量；如果输出非空，就打印一段 JSON，事件名为 UserPromptSubmit，additionalContext 写明当前的 lint 警告和输出内容。）

模式 3：团队审计日志（HTTP Hook）

场景：将所有工具使用记录发送到公司审计系统。

为什么选 HTTP Hook？审计系统通常是 REST API。Command Hook 需要额外安装 curl 并处理认证，HTTP Hook 原生支持 header 和环境变量插值。

（代码从略：在 PreToolUse 事件下配置一条 http 类型的 Hook，把工具使用记录 POST 到公司审计系统的端点，header 里的 Authorization 用 ${AUDIT_TOKEN} 做插值，并通过 allowedEnvVars 声明允许插值 AUDIT_TOKEN 这个环境变量。）

安全提示：allowedEnvVars 明确声明允许插值的环境变量。未列出的变量引用（比如 $HOME）会被替换为空字符串，防止意外暴露敏感信息。

模式 4：LLM 安全评估（Prompt Hook）

场景：在执行 Bash 命令前，用 LLM 评估命令是否安全。

为什么选 Prompt Hook？简单的模式匹配（比如 rm 加星号）无法覆盖所有危险命令。LLM 可以理解命令的语义，识别 find / -delete、dd if=/dev/zero of=/dev/sda 等非典型但危险的命令。

（代码从略：在 PreToolUse 事件下用 matcher 为 Bash 配置一条 prompt 类型的 Hook，提示词要求 LLM 评估给定的 Bash 命令在开发环境中是否可以安全执行，$ARGUMENTS 会被替换为命令详情，需要考虑是否修改系统文件、是否不可逆地删除数据、是否意外访问网络资源，超时设为 15 秒。）

模式 5：PermissionRequest 自动审批

场景：自动批准已知安全的命令模式，减少用户确认弹窗。

（代码从略：在 PermissionRequest 事件下用 matcher 为 Bash 配置一条 command Hook，直接 echo 一段 JSON，声明事件名为 PermissionRequest、decision 的 behavior 为 allow，也就是自动批准；if 条件设为 Bash(npm test*)，只对 npm test 开头的命令生效。）

模式 6：输入改写（updatedInput）

场景：强制危险命令使用安全模式。

（代码从略：在 PreToolUse 事件下用 matcher 为 Bash 配置一条 command Hook，命令先读入 stdin 传来的输入，用 jq 取出 tool_input 里的 command，再打印一段 JSON，在 permissionDecision 为 allow 的同时，通过 updatedInput 把命令改写成原命令加 --dry-run 参数；if 条件设为 Bash(rm *)，只对 rm 开头的命令生效。）

7.10 Hook 与技能/插件的协作

技能级 Hook

技能可以在 frontmatter 中定义自己的 Hook，在技能执行期间生效。这创建了层级扩展。层级关系是：全局 Hook 写在 settings.json 里，始终生效；技能级 Hook 写在技能 frontmatter 里，仅在该技能执行时生效；插件级 Hook 写在插件目录的 hooks.json 里，插件启用时生效。

例如，一个部署技能可以定义 PreToolUse Hook 来限制可执行的命令。（代码从略：这个名为 deploy 的技能在 frontmatter 中定义了 PreToolUse Hook，matcher 为 Bash，if 条件设为 Bash(rm *)，命中的话输出一段 JSON，把权限决策设为 deny，原因是 rm 命令在部署过程中被禁止。）

技能级 Hook 通过 parseHooksFromFrontmatter 解析，使用与全局 Hook 完全相同的 HooksSchema 验证。会话 Hook 按 sessionId 隔离，确保一个 Agent 的 Hook 不会泄漏到另一个 Agent。

插件 Hook 的变量替换

插件 Hook 的命令中可以使用特殊占位符。第一，${CLAUDE_PLUGIN_ROOT}，替换为插件安装目录。第二，${CLAUDE_PLUGIN_DATA}，替换为插件数据目录。第三，${user_config.keyName}，替换为用户在插件配置中设置的选项值。执行时，这些变量会被替换为实际路径，并作为环境变量注入子进程。

Hook 去重机制

当同一个 Hook 命令出现在多个配置源中时（例如用户设置和项目设置都定义了 echo hello），去重机制确保只执行一次。

去重的命名空间设计值得关注。去重键由 hookDedupKey 生成：取插件根目录或技能根目录作为前缀，都没有时用空字符串，拼上空字符分隔符，再拼上 Hook 载荷。具体有三条。第一，settings Hook 没有 pluginRoot 和 skillRoot，前缀是空字符串，所以用户、项目、本地设置里的相同命令会合并。第二，插件 Hook 的前缀是 pluginRoot，不同插件的相同模板命令（如指向 hook.sh 的那条）展开后是不同文件，不会合并。第三，技能 Hook 的前缀是 skillRoot，同理。

new Map 对于重复键保留最后一个条目，即 last-wins，对于 settings Hook 这意味着后合并的配置覆盖先合并的。

7.11 设计洞察

事件驱动加双层匹配，等于精确控制

27 种事件、matcher、if 条件三者配合，从粗粒度到细粒度层层收窄，覆盖几乎所有扩展需求。不匹配的 Hook 在 spawn 进程之前就被过滤——这是真正的零成本抽象。

退出码是最核心的通信协议

0、1、2 三个退出码的区分看似简单，实则精妙。它把信息传递分成三个级别：退出码 0 是静默成功，1 是告知用户，2 是要求模型处理。这让 Hook 作者不需要学习复杂的 JSON 协议就能控制基本行为——exit 2 比构建 JSON 对象简单得多。

异步唤醒是创新设计

传统的 Hook 系统要么同步阻塞、慢，要么异步 fire-and-forget、没有反馈。asyncRewake 是第三种模式——后台运行加按需中断，退出码 2 的约定让长时间运行的检查只在失败时中断模型，完美平衡了性能和反馈。

多层性能优化

Hook 系统在热路径上有三层优化。第一，hasHookForEvent：大多数事件无 Hook 配置，快速短路返回。第二，matcher 和 if 前置过滤：不匹配的 Hook 不 spawn 进程。第三，Callback 快速路径：匹配集合全为 internal 为 true 的 callback 时，跳过 JSON 序列化和进度事件，省下约 70% 开销；这些内部 Hook，比如文件访问跟踪、commit 归因，在每次工具调用时都触发。

安全设计遵循"全或无"原则

历史漏洞证明"大部分 Hook 需要信任"不够，必须是"所有 Hook 都需要信任"。配置快照、信任检查、环境变量白名单、CRLF 注入防护——每一层都假设攻击者可以控制前一层的输入。

Hook 是 Agent Loop 的横切关注点

Hook 系统相当于 Agent Loop 的 AOP，也就是面向切面编程层。它不修改核心循环的任何逻辑，而是在关键节点注入横切逻辑。这使得核心循环保持简洁，扩展能力通过 Hook 系统实现。从架构上看，Hook 系统是连接 Claude Code 内部机制和外部生态——企业基础设施、CI/CD、安全策略——的桥梁。

上一章是记忆系统，下一章是多 Agent 架构。
