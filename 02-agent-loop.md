---
title: 第 2 章：系统主循环（朗读版）
---

第 2 章：系统主循环

本文是 docs/02-agent-loop.md 的朗读版，表格与代码已转为口语描述，内容未增删。

这是整个 Claude Code 最核心的章节。理解了主循环，就理解了 Claude Code 的灵魂。

2.1 全景：一次完整的交互

当用户输入一条消息时，Claude Code 执行以下流程：先是用户输入和上下文组装，然后模型做出决策，接着执行工具，把结果注入上下文，最后决定继续还是停止。

这个循环不断重复，直到模型决定不再调用工具，返回纯文本响应为止。这就是 Agent Loop，也就是代理循环。

2.2 双层生成器架构

Claude Code 的查询系统采用双层生成器架构，清晰分离会话管理与查询执行。

这张架构图的流程是：首先，外层的 QueryEngine 接收 submitMessage 调用，交给 processUserInput 处理用户输入，然后发起 query 调用，进入内层的 query 函数。在 query 内部，先做消息规范化和压缩，再发起 API 流式调用，随后执行工具，最后判断继续还是终止。如果要继续，就回到消息规范化这一步，如此循环往复。

这套架构可以从四个维度来对比。第一，作用域：QueryEngine 覆盖对话的全生命周期，query 只覆盖单次查询循环。第二，状态：QueryEngine 的状态是持久化的，包括 mutableMessages 和 usage。query 的状态在循环内部，是一个 State 对象，每次迭代重新赋值。第三，预算追踪：QueryEngine 负责每一轮的 USD 检查和结构化输出重试。query 负责 Task Budget 跨压缩结转和 Token 预算续写。第四，恢复策略：QueryEngine 处理权限拒绝和孤儿权限。query 处理 PTL 排水与压缩，以及 max_output_tokens 的升级和重试。

为什么要分两层？因为会话管理和查询执行的关注点完全不同。QueryEngine 关心的是“用户说了什么、花了多少钱、这轮结果是否成功”；query 关心的是“消息是否需要压缩、API 返回了什么、工具执行是否成功、是否需要恢复”。双层分离使得每层的代码都更聚焦、更容易测试。

2.3 QueryEngine：会话生命周期管理

src/QueryEngine.ts 共 1295 行，是对话的外壳。它的核心方法 submitMessage 驱动一次完整的用户交互。

核心配置参数（节选）

QueryEngine 通过 QueryEngineConfig 接收配置，核心字段如下。

（代码从略：这段代码定义了 QueryEngineConfig 类型，要点是它集中声明了引擎运行所需的全部配置。必填部分包括工具执行的工作目录 cwd、包含 40 多个内置工具的工具集 tools、斜杠命令 commands、活跃的 MCP 服务端连接 mcpClients、来自 .claude/agents/ 的自定义 Agent 定义 agents，以及权限判定函数 canUseTool 和读写界面状态的 setAppState 等方法。可选部分包括会话恢复的初始消息、用于去重读取的文件状态缓存 readFileCache、完全覆盖或追加系统提示词、模型覆盖与错误降级模型、扩展思考配置、最大工具调用轮次 maxTurns、USD 成本上限 maxBudgetUsd、API 侧 Token 预算 taskBudget、结构化输出的 JSON Schema、取消控制器和孤儿权限处理等。另有一批面向 SDK 的字段从略。）

几个值得注意的设计细节：

- canUseTool 包装：submitMessage 内部会包装这个函数，在原有权限检查基础上追踪所有权限拒绝事件。这些拒绝记录最终会在结果消息中返回给 SDK 消费者，比如桌面应用，让它们知道用户拒绝了哪些操作。
- readFileCache：避免重复回传同一个文件的全文。如果模型在第 3 轮调用 FileReadTool 读了 src/query.ts，第 5 轮再次请求同一文件的同一范围，且文件在磁盘上未改动，这时仍会通过 stat 校验 mtime，返回的是一个 file_unchanged 存根，而不是重发全文。那次读取的结果还留在上下文里，重发只会白白重复消耗 cache_creation token。
- orphanedPermission：处理一种边缘情况。上一次会话在用户授权“始终允许 BashTool”后崩溃，权限没有持久化，下次启动时会把这个“孤儿权限”重放一次。

submitMessage 八阶段生命周期

submitMessage 驱动一次完整的用户交互，分为 8 个阶段。

这张流程图可以概括为：首先，submitMessage 依次经历设置、孤儿权限处理和用户输入处理三个阶段，完成清除技能发现、包装 canUseTool、初始化模型配置、加载系统提示词、解析斜杠命令与附件，并把消息持久化。然后产出一条系统初始化消息，说明工具和命令的注册信息，再判断这是不是本地命令：如果是，就直接产出本地命令输出并提前返回；如果不是，就进入主查询循环。接着在循环结束后做预算检查，USD 超限或结构化输出重试达到 5 次都会报错。最后提取结果，检查 isResultSuccessful，产出最终的结果消息。

逐阶段展开：

阶段 1，设置。为什么每轮开头都要清空技能发现追踪集合？这个集合记录本轮发现过的技能名，用于喂给 tengu_skill_tool_invocation 遥测里的 was_discovered 标记。它需要跨一次 submitMessage 内的两次 processUserInputContext 重建存活，但每次 submitMessage 开头都清空，是为了避免 SDK 模式下多轮会话里这个集合无限增长。

阶段 2，孤儿权限。它只在会话的第一次 submitMessage 调用时触发，且只触发一次，orphanedPermission 用过即清空。这处理的是上一个会话崩溃后遗留的权限授权。

阶段 3，用户输入处理。processUserInput 是一个复杂的函数，它需要解析斜杠命令，比如 /compact 触发手动压缩，/memory 管理记忆；需要处理附件，包括图片、PDF 和文件引用；最后将处理后的消息推入 mutableMessages 并持久化到磁盘。

阶段 5，本地命令检查。像 /clear 这样的命令不需要调用 API，它们只是清理本地状态。如果 processUserInput 把 shouldQuery 设为 false，就直接产出命令输出并提前返回，跳过整个查询循环。

阶段 6，主查询循环。这是最复杂的阶段。程序用 for await 迭代查询生成器，用 switch 按消息类型分派 9 类消息：

- tombstone：消息删除控制信号，跳过不处理。
- assistant 消息：推入消息列表，并 yield 给上层。
- progress 消息：行内进度记录。
- user 消息：工具结果注入。
- stream_event：流式事件，内含 message_start、message_delta、message_stop，用于更新 Token 使用统计。
- attachment：提取结构化输出 StructuredOutput，并承载 max_turns_reached 终止信号，收到即 yield 一条 error_max_turns 结果并直接返回。
- stream_request_start：请求开始信号，跳过不处理。
- system：有两个子类型，compact_boundary 触发 snip、splice、GC 清理；api_error 则 yield 重试信号。
- tool_use_summary：工具使用摘要。

阶段 7，预算检查。这里有两种预算限制：一是 USD 成本，触线条件是 getTotalCost 大于等于 maxBudgetUsd；二是结构化输出重试次数，最多 5 次。

阶段 8，结果提取。isResultSuccessful 检查最后一条 assistant 消息是否有效。最终 yield 的结果消息带一整套元数据：usage 是 Token 使用量，cost 是 USD 成本，turns 是工具调用轮次，另有 stop_reason 和 permission_denials，也就是用户拒绝过的权限，等字段。

2.4 query 函数：核心循环的实现

src/query.ts 共 1729 行，是 Claude Code 最复杂的单个模块，实现了一个基于状态机的异步生成器循环。

核心签名

（代码从略：这段代码展示了 query 的函数签名，要点是它是一个异步生成器函数，接收 QueryParams 参数，依次产出流式事件、请求开始事件、各类消息、墓碑消息和工具使用摘要，最终返回一个 Terminal 终止值。）

关键点：这是一个 async function*，也就是异步生成器。它不是一次性返回结果，而是边执行边 yield 事件，使调用方可以实时渲染流式输出。

循环状态

每次循环迭代共享一个可变的 State 对象。

（代码从略：这段代码定义了 State 类型，要点是它承载循环的全部可变状态，包括当前消息列表 messages、工具执行上下文 toolUseContext、自动压缩追踪状态、输出 Token 恢复计数、是否已尝试反应式压缩、输出 Token 上限覆盖值、待处理的工具使用摘要、Stop Hook 激活标志、当前轮次 turnCount，以及上一次循环继续的原因 transition。）

不可变参数与可变状态

query 内部有一个重要的设计区分。

（代码从略：这段代码展示了 queryLoop 函数的开头，要点是把系统提示词、用户上下文、系统上下文、canUseTool、降级模型、查询来源、最大轮次、是否跳过缓存写入这些参数从 params 中解构出来，循环期间永不重新赋值。而可变的跨迭代状态放在一个 State 对象里，由 7 个继续点通过整体赋值来更新。）

params 中的字段在循环期间是常量；state 在每个 continue site 通过整体赋值更新，而不是逐字段修改。这让状态变更更加明确和可追踪。

单次循环迭代流程

这张流程图描述了单次迭代：首先，进入循环后，消息列表先经过 Tool Result 预算裁剪，然后依次做 History Snip 剪裁、Microcompact 微压缩加缓存工具结果去重、Context Collapse 上下文折叠，以及 Token 达到阈值时触发的 Autocompact 自动全量压缩。接着构建 API 请求，拼接系统提示、工具列表和消息，流式调用 callModel 并创建 StreamingToolExecutor，在收集流式响应的同时执行工具，工具执行完再注入附件，包括记忆召回和技能发现。最后检查停止条件：没有工具调用就终止循环，返回 Terminal；有工具调用就设置 transition 为 next_turn，继续下一轮；遇到 PTL 错误就进入恢复机制，再回到循环入口。

循环体代码走读

跟着代码走一遍循环体的关键步骤。

第一步：4 级压缩流水线，详见第 3 章。

每次循环迭代的入口处，消息列表依次经过 Tool Result Budget、Snip、Microcompact、Context Collapse 和 Autocompact。这是防御性设计，即使上一轮工具返回了 100K Token 的输出，压缩流水线也会在 API 调用前将其控制在预算内。

（代码从略：这段代码依次执行 4 级压缩，要点是先做 Tool Result 预算裁剪；再在 HISTORY_SNIP 开关开启时执行 History Snip 剪裁，并记录释放的 Token 数；然后做 Microcompact 微压缩；最后在 CONTEXT_COLLAPSE 开关开启时执行上下文折叠。）

第二步：构建 API 请求

（代码从略：这段代码很短，要点是用 asSystemPrompt 构建完整的系统提示词，把系统上下文追加在系统提示词末尾，而用户上下文则通过 prependUserContext 前置于消息。）

上下文的注入顺序对提示词缓存有影响：系统提示词相对稳定，Git 状态这类系统上下文追加在它末尾；CLAUDE.md、日期这类用户上下文则前置于消息。这种安排让系统提示词部分更容易命中缓存。

第三步：流式调用与工具并行执行

callModel 返回一个异步生成器，StreamingToolExecutor 在流式接收响应的同时，就开始执行已完成的工具调用，详见 2.4.1 节。

第四步：记忆预取消费

（代码从略：这段代码只有几行，要点是在循环入口用 using 关键字创建记忆预取 pendingMemoryPrefetch，确保在所有退出路径上自动释放资源。）

using 是 TypeScript 的显式资源管理语法。当生成器退出时，无论正常返回还是异常，pendingMemoryPrefetch 的 Symbol.dispose 都会自动调用，用于发送遥测和清理资源。记忆预取在模型流式生成期间并行运行，通过 settledAt 守卫确保每轮只消费一次。

2.4.1 流式处理与并行工具执行

Claude Code 对流式响应的用法不止于渲染。它利用 StreamingToolExecutor 实现了流式工具并行执行，在模型还在生成后续 token 时，已经完成解析的工具调用就立即分发执行。这是 query 循环内部的关键优化环节。

这张图分上下两部分。上半部分是 StreamingToolExecutor 的内部结构：API 流式输出不断到达，每当一个 tool_use 解析完成就立即执行，结果随即就绪，而模型此时还在继续生成后续内容，图中三个工具都是这样交错推进的。下半部分是时间线对比：串行执行要等 API 全部结束，再依次跑完三个工具；流式并行则利用 5 到 30 秒的流式窗口，让工具在模型生成期间就铺开执行，覆盖每个工具约 1 秒的延迟，结果即时可用。

StreamingToolExecutor 实现原理

StreamingToolExecutor 位于 src/services/tools/StreamingToolExecutor.ts，共 530 行，核心是一个带并发控制的工具执行队列。每个工具沿 queued、executing、completed、yielded 四个状态流转。

（代码从略：这段代码是核心调度逻辑，要点有三。第一，每解析完一个 tool_use block 就立即入队，并马上尝试调度执行。第二，并发控制：canExecuteTool 判断当前没有工具在执行时放行，或者新工具与所有正在执行的工具都是并发安全的才允许并行，否则必须独占。第三，流式结束后收割所有已完成的结果，把状态标记为 yielded 并逐条产出消息更新。）

关键设计细节：

1. addTool：API 流式响应在解析到完整的 tool_use JSON block 时调用此方法。注意是“完整的 block”，不需要等整个 API 响应结束，一个 tool_use block 的 JSON 完成解析就可以分发执行。
2. 并发安全分类：每个工具通过 isConcurrencySafe 声明是否可以并行。读文件、搜索等只读操作标记为并发安全，可以同时执行；写文件、Bash 命令等标记为非并发，必须独占执行。
3. Bash 错误级联：当一个 Bash 工具出错时，siblingAbortController 会取消所有正在并行执行的兄弟工具，因为 Bash 命令之间经常有隐式依赖，比如 mkdir 失败后，后续命令就没有意义了。但读文件、搜索等独立操作的失败不会触发级联。
4. getCompletedResults 与 getRemainingResults：前者非阻塞地收割已完成结果，后者异步等待剩余执行中的工具。两者配合，实现了“流式期间即时收割，流式结束后等待尾部”的模式。

这种设计的效果是：在一个典型的 API 响应，也就是 5 到 30 秒的流式窗口中，多个工具就能分发并执行完毕。到流式结束时，工具结果已经可用，消除了串行执行的瓶颈。

2.6 Feature Flag 条件加载

query.ts 使用了 6 个 Feature Flag 条件加载模块，分散在文件头部的三个 eslint-disable 块中。其中前 4 个与上下文压缩和工具系统密切相关，是理解主循环的核心；后 2 个，jobClassifier 和 taskSummaryModule，分别服务于模板分类和后台会话摘要，属于辅助功能。

（代码从略：这段代码把 6 个模块按三组做条件加载，要点是用 feature 函数判断开关，开启时才 require 对应模块，否则置为 null。第一组是上下文压缩相关的 reactiveCompact 与 contextCollapse。第二组是技能搜索与模板分类相关的 skillPrefetch 与 jobClassifier。第三组是历史剪裁与后台会话摘要相关的 snipModule 与 taskSummaryModule。）

这 6 个开关对应的变量名和功能如下。第一，REACTIVE_COMPACT 对应变量 reactiveCompact，负责 PTL 错误时的反应式全量压缩。第二，CONTEXT_COLLAPSE 对应 contextCollapse，实现投影式上下文折叠。第三，EXPERIMENTAL_SKILL_SEARCH 对应 skillPrefetch，负责技能搜索预取。第四，TEMPLATES 对应 jobClassifier，是任务模板分类器。第五，HISTORY_SNIP 对应 snipModule，负责历史消息 snip 剪裁。第六，BG_SESSIONS 对应 taskSummaryModule，负责后台会话任务摘要生成。

这个模式有三个层次：

1. 编译时消除：feature 在 Bun bundler 构建时求值。外部构建中 REACTIVE_COMPACT 这类开关返回 false，整个 require 分支被 tree-shaking 移除。
2. 类型安全：as typeof import 让 TypeScript 知道模块的完整类型，IDE 补全和类型检查不受影响。
3. 运行时守卫：代码中先用 if 判断 contextCollapse 存在，再调用它的方法，编译时也会把这个 null 检查一并消除。

2.7 七个继续点（Continue Sites）

query 循环有 7 个导致循环继续的位置，每个对应一种恢复策略。

第一，next_turn，触发条件是模型调用了工具，处理方式是正常继续，带上工具结果。第二，collapse_drain_retry，触发条件是 PTL 错误且 Context Collapse 有暂存，处理方式是提交折叠、释放 Token、重试。第三，reactive_compact_retry，触发条件是 PTL 错误且折叠还不够，处理方式是强制全量摘要压缩后重试。第四，max_output_tokens_escalate，触发条件是输出 Token 不够，处理方式是升级到 64K Token 限制。第五，max_output_tokens_recovery，触发条件是升级不可用或已经用过，处理方式是注入续写提示，最多重试 3 次。第六，stop_hook_blocking，触发条件是 Stop Hook 阻止终止，处理方式是继续执行。第七，token_budget_continuation，触发条件是 Token 预算续写，处理方式是继续生成。

PTL，即 Prompt Too Long，恢复流程

首先，PTL 错误发生后进入第一阶段，Context Collapse 排水，调用 recoverFromOverflow 提交暂存的折叠。然后检查是否释放了 Token：如果释放了，就以 collapse_drain_retry 为标记重试 API 调用；如果没有，进入第二阶段反应式压缩，调用 tryReactiveCompact 强制全量摘要压缩。压缩成功就以 reactive_compact_retry 为标记重试；仍然失败就产出错误，返回 prompt_too_long。

Max Output Tokens 恢复

首先，出现 max_output_tokens 错误后，先判断能否升级。如果能，就升级到 ESCALATED_MAX_TOKENS 的 64K 限制，不注入用户消息，直接重试。如果不能，再检查重试次数是否小于 3：小于 3 就注入一条 meta 用户消息，告诉模型输出已达上限，请直接从断点续写，然后重试；达到 3 次就产出那条扣留的错误。

2.8 错误扣留策略（Withholding）

这是 Claude Code 里相当巧妙的一处设计：可恢复的错误不立即 yield 给上层。

工作原理

当出现 prompt_too_long 或 max_output_tokens 错误时，query 不会立即通知调用方。它将错误推入 assistantMessages 但保留引用，然后运行恢复检查。如果恢复成功，错误永远不会暴露给调用者，包括 SDK 消费者和桌面应用，用户完全感知不到中间的错误。

（代码从略：这段代码定义了错误扣留的检测函数 isWithheldMaxOutputTokens，要点是判断消息类型为 assistant，且 apiError 为 max_output_tokens，以此识别被扣留的输出超限错误。）

一个实际场景

假设模型正在编辑一个大文件，生成了 16000 Token 的输出后被 max_output_tokens 截断：

1. 错误发生：API 返回 stop_reason 为 max_output_tokens。
2. 扣留而非暴露：错误包装成一条 AssistantMessage，其 apiError 为 max_output_tokens，推入消息列表，但不 yield 给调用方。
3. 恢复策略一，升级：检查是否可以升级到 ESCALATED_MAX_TOKENS 的 64K。如果可以，直接用更大的 Token 限制重试，不注入任何用户消息。
4. 恢复策略二，续写：如果升级不可用或已经用过，注入一条 meta 用户消息，告诉模型输出已达到上限，请直接继续，不要道歉，也不要复述之前做了什么；如果截断正好发生在思路中间，就从中间接着说，并把剩余工作拆成更小的步骤。这样让模型从断点继续，最多重试 3 次。
5. 成功恢复：如果恢复成功，那条扣下的错误消息永远不会 yield，SDK 消费者比如桌面应用看不到任何错误，用户感知到的是一次流畅的响应。

只有当所有恢复尝试都失败时，也就是升级不可用，加上 3 次续写都失败，错误才 yield 给上层。

为什么这么设计？

如果不做扣留，SDK 消费者，包括桌面应用和 Bridge 模式，收到 error 类型的消息后会终止会话，即使后端的恢复循环还在运行，前端已经不再监听了。扣留机制确保前端只看到“干净”的结果流。

2.9 Token 使用追踪

QueryEngine 的 totalUsage 初始化为 EMPTY_USAGE，这个常量定义在 QueryEngine.ts 的第 206 行，用于维护完整的 Token 使用统计。

（代码从略：这段代码定义了 EMPTY_USAGE 常量，要点是把所有使用计数归零，包括输入 Token、缓存创建与缓存读取的输入 Token、输出 Token，以及网页搜索和网页抓取的请求次数。另外还有服务层级、一小时与五分钟两档的缓存创建计数、推理地域、迭代记录和速度等字段。）

追踪机制：

- 每条 API 响应的 message_delta 事件都会更新 currentMessageUsage。
- message_stop 时，currentMessageUsage 通过 accumulateUsage 累加到 totalUsage。
- getTotalCost 基于 totalUsage 和模型定价计算 USD 总成本。
- 一旦 getTotalCost 大于等于 maxBudgetUsd，整个查询终止，这是防止意外高成本的安全机制。

cache_read_input_tokens 和 cache_creation_input_tokens 的追踪对提示词缓存策略至关重要，它们告诉系统缓存是否在有效工作。缓存断裂检测对应 promptCacheBreakDetection.ts，就依赖这些数据来判断是否发生了缓存失效。

2.10 停止条件

循环在以下条件下终止：

1. 模型未调用工具：返回纯文本响应，正常结束。
2. 达到最大轮次：受 maxTurns 限制。
3. USD 预算超限：getTotalCost 大于等于 maxBudgetUsd。
4. 用户中断：abortController 的 signal 触发。
5. 不可恢复的错误：PTL 和 MOT 恢复全部失败。

设计决策：连续压缩失败的熔断器为什么阈值是 3 次？

MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES 等于 3，定义在 src/services/compact/autoCompact.ts。源码注释引用了生产数据：BQ 2026 年 3 月 10 日的统计显示，1279 个会话出现了 50 次以上的连续失败，单个会话最多达到 3272 次，全球每天浪费约 250K 次 API 调用。没有这个熔断器之前，压缩一旦进入失败循环，会在每一轮都白白发起一次完整的摘要 API 调用。这种调用的输出上限是 MAX_OUTPUT_TOKENS_FOR_SUMMARY，即 20K Token，并非每次都真的消耗这么多。注意熔断器不会直接终止主循环：连续失败 3 次后，checkAutoCompact 对本会话后续的 autocompact 尝试直接返回 wasCompacted 为 false，压缩变成空操作；query 拿到失败计数只更新追踪状态，随即继续往下迭代。如果上下文最终仍然超限，要经由上面第 5 条，也就是 PTL 和 MOT 恢复全部失败，才真正终止。3 次阈值在“给压缩服务恢复机会”和“避免资源浪费”之间取得平衡。

设计决策：为什么用异步生成器而不是回调或事件？

query 是一个 async function*，通过 yield 逐步输出事件。相比 EventEmitter 这类回调模式，生成器有两个关键优势。一是背压控制：消费端不处理完上一个事件，生产端不会继续执行，天然防止事件堆积。二是线性控制流：循环的 7 个 continue site 可以用普通的 state 整体赋值加 continue 来表达，不需要状态机的显式转换表。代价是调用方必须用 for await of 来消费，但在 Claude Code 中只有 QueryEngine 是消费者，这个约束完全可接受。

2.11 设计亮点总结

1. 双层生成器分离关注点：QueryEngine 管会话生命周期，query 管单次循环。
2. 流式工具并行执行：利用 API 流式窗口覆盖工具延迟。
3. 错误扣留保证用户无感知恢复：可恢复错误不暴露给上层。
4. 7 个精确的继续点：每种恢复策略都有明确的 transition 标记，可测试、可追踪。
5. 编译时 Feature Gate：内部功能从外部构建中物理移除。
6. Task Budget 跨压缩结转：压缩前后的 Token 预算无缝衔接。

动手实践：在 claude-code-from-scratch 项目的 src/agent.ts 中，你可以看到一个约 1500 行的 Agent 主循环实现。对比本章描述的双层生成器架构，思考两个问题：为什么最小实现不需要分两层？什么规模下才值得引入 QueryEngine 这样的会话管理层？参见教程第 1 章：Agent Loop。

上一章是概述，下一章是上下文工程。
