---
title: 第 9 章：Plan 模式（朗读版）
---

第 9 章：Plan 模式

本文是 docs/10-plan-mode.md 的朗读版，表格与代码已转为口语描述，内容未增删。

先想清楚再动手——Plan 模式是 Claude Code 中唯一一个主动降低自身权限来换取用户信任的机制。

9.1 为什么需要 Plan 模式

想象这样一个场景：你让 Claude Code “重构整个认证系统”。它二话不说就开始改文件——改了 12 个文件、删了 3 个函数、引入了一个你完全不想要的 JWT 库。你只能 git checkout . 然后重来。

这就是没有 Plan 模式的世界。

Plan 模式的设计理念是：对于复杂任务，让模型先探索、再规划，用户审批后才动手。它通过权限降级，也就是禁止一切写操作，强制模型进入“只读探索加输出计划”的工作模式。直到用户明确批准计划后，才恢复写权限。

关键文件有七个。第一个是 src/tools/EnterPlanModeTool/EnterPlanModeTool.ts，共 127 行，职责是进入 Plan 模式的工具。第二个是 src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts，共 493 行，职责是退出 Plan 模式加审批流程。第三个是 src/utils/plans.ts，共 398 行，职责是计划文件管理，包括 slug、读写和恢复。第四个是 src/utils/planModeV2.ts，共 96 行，职责是配置，包括 Agent 数量和实验变体。第五个是 src/utils/messages.ts 的第 3136 到 3417 行，约 280 行，职责是 Plan 模式系统消息生成。第六个是 src/utils/permissions/permissionSetup.ts，约 60 行，职责是权限上下文准备与恢复。第七个是 src/bootstrap/state.ts 的第 1349 到 1470 行，约 120 行，职责是 Plan 模式全局状态。

9.2 全景：一次完整的 Plan 模式流程

先把整个流程走一遍。首先，用户向模型发出重构认证系统的请求。模型调用 EnterPlanMode 进入 Plan 模式。状态机执行从 default 到 plan 的模式转换，并调用 prepareContextForPlanMode 做上下文准备，然后向模型确认已进入 plan 模式。

接着，系统注入 plan_mode 附件，也就是五阶段工作流指令。模型依次执行五个阶段。第一阶段，启动 Explore Agent 探索代码。第二阶段，启动 Plan Agent 设计方案。第三阶段，审阅各 Agent 的结果，并向用户提问。第四阶段，把计划写入用户目录下的计划文件，例如 bold-eagle.md。第五阶段，调用 ExitPlanMode 提交计划。

然后，状态机向用户弹出审批对话框，展示计划内容。用户可以批准，也可以先编辑计划再批准。

最后，状态机恢复进入前的模式，并把已退出 Plan 模式的标志置为真。审批后的完整计划文本会返回给模型。系统再注入 plan_mode_exit 附件，告诉模型已退出 plan 模式，可以开始实施。模型随即按计划开始编辑文件。

整个流程的关键在于状态转换的对称性：进入时记住原模式，也就是 prePlanMode，退出时精确恢复。这保证了 Plan 模式是一个“可嵌套的插入层”。无论你原来是 default、auto 还是 bypassPermissions 模式，Plan 模式结束后都能无缝回到之前的状态。

9.3 进入 Plan 模式：两条路径

有两种方式进入 Plan 模式，但最终都汇聚到同一个状态转换函数。两条路径是这样的：一条是用户主动，输入 /plan 命令；另一条是模型主动，调用 EnterPlanMode 工具。两者都汇聚到 handlePlanModeTransition，再调用 prepareContextForPlanMode 准备上下文，最后通过 setMode 把模式设置为 plan。

9.3.1 用户主动：/plan 命令

用户在 REPL 中输入 /plan 或 /plan 重构认证系统，触发的是 src/commands/plan/plan.tsx 第 64 到 121 行的逻辑。（代码从略：这段代码先读取当前模式。如果当前不是 plan 模式，就调用 handlePlanModeTransition 做状态转换，再调用 prepareContextForPlanMode 准备权限上下文，最后把模式设置为 plan 并写回应用状态。）

如果命令带了描述，比如 /plan 重构认证系统，会同时将描述作为用户消息提交给模型，触发一次完整的查询循环。模型在 plan 模式下收到这条消息后，就会开始探索。

9.3.2 模型主动：EnterPlanMode 工具

这是更常见的路径。模型判断当前任务复杂度较高时，主动调用 EnterPlanMode 工具请求进入计划模式。

相关代码在 src/tools/EnterPlanModeTool/EnterPlanModeTool.ts 第 77 到 101 行。（代码从略：这段代码是工具的 call 方法。它首先检查上下文里有没有 agentId，如果有就抛出错误，因为子 Agent 不允许进入 plan 模式，plan 是用户级别的决策。然后读取当前模式，调用 handlePlanModeTransition，并在调用 prepareContextForPlanMode 之后把模式设置为 plan。最后返回一条消息，提示已进入 plan 模式，模型接下来应专注于探索。）

注意 context.agentId 这个检查——这是一条关键的设计约束：子 Agent 不能进入 Plan 模式。因为 Plan 模式需要用户交互来审批计划，而子 Agent 运行在后台，没有与用户直接交互的能力。一旦允许子 Agent 进入 Plan 模式，它会永远卡在等待审批的状态。

9.3.3 工具的 Prompt：引导模型何时进入

模型怎么知道什么时候该进入 Plan 模式？答案在工具的 prompt 里。Claude Code 用精心设计的 prompt 引导模型判断。

对于外部用户，也就是 src/tools/EnterPlanModeTool/prompt.ts 第 16 到 99 行，prompt 列出了 7 种应该进入 Plan 模式的条件。第一种，新功能实现，即添加有意义的新功能。第二种，多种可行方案，任务有多种合理解法。第三种，代码修改，即影响现有行为或结构的变更。第四种，架构决策，需要在模式或技术之间做选择。第五种，多文件变更，可能涉及 2 到 3 个以上文件。第六种，需求不明确，需要先探索才能理解范围。第七种，用户偏好重要，实现可以有多种合理方向。

而对于内部用户，也就是 ant，prompt 要更保守——只在真正存在架构歧义时才建议进入 Plan 模式，避免过度规划拖慢节奏。这反映了一个实际观察：内部用户通常对代码库更熟悉，不需要那么多“先规划再动手”的保护。

9.4 Plan 模式下的系统消息注入

进入 Plan 模式后，Claude Code 通过附件系统，英文叫 Attachment System，在每轮对话中注入指令，告诉模型“你现在只能读不能写”，以及“应该按什么流程工作”。

9.4.1 附件的节流机制

不是每轮都注入完整指令——那会浪费大量 token。Plan 模式的系统消息是模型行为的防护栏，即“你现在只能读不能写”。但每轮都重复完整指令，不仅浪费 token，还可能让模型对这些指令生出“疲劳”，过于频繁的提醒反而削弱效力。所以 Claude Code 采用渐进式提醒策略：首轮提供完整上下文，建立规则意识；中间若干轮信任模型的短期记忆；之后定期用轻量提醒，刷新模型对当前模式的认知。

src/utils/attachments.ts 第 1189 到 1242 行实现了这套节流逻辑，具体规则用口语描述就是：第 1 轮注入完整指令，即 full 版本，因为模型首次进入，需要完整上下文。第 2 到第 4 轮不注入，以节省 token，模型应该还记得。第 5 轮注入简短提醒，即 sparse 版本，防止模型忘记自己在 plan 模式。第 6 到第 9 轮不注入，继续节省。第 10 轮再次注入简短提醒。之后每 25 轮注入一次完整指令，在长会话中完整刷新上下文。

配置常量在 src/utils/attachments.ts 第 259 到 262 行。（代码从略：这段代码导出了 PLAN_MODE_ATTACHMENT_CONFIG 常量，包含两项配置。TURNS_BETWEEN_ATTACHMENTS 为 5，表示每 5 轮注入一次。FULL_REMINDER_EVERY_N_ATTACHMENTS 为 5，表示每 5 次注入中有 1 次是完整版。）

这意味着完整指令大约每 25 轮，也就是 5 乘 5，才出现一次。其余时候用极短的 sparse 提醒，维持模型对当前模式的意识，每条约 300 字符。

9.4.2 两种工作流模式

Claude Code 实际上有两套完全不同的 Plan 模式工作流，通过 feature gate 切换。

5 阶段工作流（默认）

这是大多数用户看到的模式，位于 src/utils/messages.ts 第 3207 到 3297 行。注入的系统消息将整个规划过程分为 5 个严格的阶段。第一阶段叫 Initial Understanding，即初始理解，启动 Explore Agent 探索代码库，这个阶段只能使用 Explore 子 agent。第二阶段叫 Design，即设计，启动 Plan Agent 设计方案，可以并行启动多个 Agent 从不同角度设计。第三阶段叫 Review，即审阅，综合各 Agent 的结果，向用户提问澄清。第四阶段叫 Final Plan，即最终计划，将最终方案写入计划文件。第五阶段是调用 ExitPlanMode，提交计划等待用户审批。

Phase 2 的 Plan Agent 数量随订阅类型变化，见 src/utils/planModeV2.ts 第 5 到 29 行。Phase 1 的 Explore Agent 数量则固定为最多 3 个，与订阅无关——getPlanModeV2ExploreAgentCount 恒返回 3，只有环境变量 CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT 能覆盖。（代码从略：这段代码是 getPlanModeV2AgentCount 函数，环境变量覆盖优先。如果订阅类型是 max，且额度档位是 default_claude_max_20x，返回 3 个并行 Plan Agent。如果是 enterprise 或 team 用户，也返回 3 个。其他用户返回 1 个。）

这么设计是因为 Plan Agent 会消耗大量 token——每个 Agent 都要独立探索代码库，只有高额度用户才能承受 3 个 Agent 并行规划的成本。

迭代工作流（Interview Phase）

这是一种更灵活的替代方案，位于 src/utils/messages.ts 第 3323 到 3383 行，通过 isPlanModeInterviewPhaseEnabled 控制开启。它的区别在于不分阶段，而是一个持续循环。先探索，用只读工具读代码；然后更新计划文件，每次发现都立即写入计划；遇到歧义就问用户；之后回到探索，如此循环，直到计划完成。

关键的 prompt 差异体现在对话风格上。5 阶段模式要求模型最多并行启动 3 个 Explore agent，鼓励一次性大量探索，然后综合。迭代模式要求模型先快速浏览几个关键文件，写出一个骨架计划，并向用户提出第一轮问题，不要在与用户充分交流之前就做穷尽式探索，鼓励快速交互、渐进式深入。

9.4.3 Phase 4 的四种实验变体

Phase 4 定义最终计划的格式要求，目前是一个正在进行的 A/B 实验，实验名 tengu_pewter_ledger，四个变体见 src/utils/messages.ts 第 3156 到 3205 行。

第一种变体叫 control，是基线，采用完整格式，包括 Context、推荐方案、文件路径和验证步骤。第二种叫 trim，目标是中等压缩，只保留一行的 Context 和单个验证命令。第三种叫 cut，目标是大幅压缩，禁止写 Context 段落，每个文件只占一行，只用单个验证命令。第四种叫 cap，目标是极致压缩，禁止一切 prose，每个文件只写一个条目，并硬限制 40 行。

这个实验的动机来自生产数据：基线，也就是 control，之下的计划文件，p50 为 4906 字符，p90 为 11617 字符，均值 6207 字符。Opus 输出价格是输入的 5 倍，过长的计划文件直接推高了成本。而且拒绝率与计划长度正相关：小于 2K 的计划拒绝率是 20%，大于 20K 的拒绝率高达 50%。也就是说，计划越长用户越不满意。

9.5 计划文件管理

9.5.1 文件命名与存储

每个会话的计划文件存储在 ~/.claude/plans/ 目录下，文件名是一个随机生成的 word slug。例如，主会话的计划是 bold-eagle.md，子 Agent 7 的计划是 bold-eagle-agent-7.md。

src/utils/plans.ts 第 32 到 49 行的逻辑如下。（代码从略：这段代码是 getPlanSlug 函数，根据会话 ID 返回 slug。它先查缓存；缓存没有就生成一个新的 word slug，最多重试 10 次以避免文件名冲突，直到对应文件不存在为止；然后写入缓存并返回。）

Slug 在会话内缓存，缓存名叫 planSlugCache，是一个从会话 ID 到字符串的映射，确保同一会话始终写入同一个文件。为什么用 word slug 而不是 UUID？因为用户可能需要手动打开和编辑这个文件——bold-eagle.md 比 a3f7b2c1-4d5e-6f78.md 更易于识别和记忆。

9.5.2 Resume 与 Fork

当用户恢复之前的会话时，需要找回对应的计划文件。之所以需要多达 5 层恢复策略，根本原因在于本地会话和远程会话，即 CCR，的文件持久化能力完全不同。本地用户的计划文件安全地存储在磁盘上，恢复很简单。远程用户的 pod 随时可能被回收，文件可能已经不存在，必须从 transcript 中的各种位置尝试恢复。

copyPlanForResume 函数位于 src/utils/plans.ts 第 164 到 230 行，它的恢复策略是分层的。第一层，直接读取文件，适用于文件还在磁盘上的情况，这是本地会话的常见情况。第二层，文件快照恢复，从 transcript 中的 file_snapshot 消息恢复，用于远程会话。第三层，ExitPlanMode 输入，从 tool_use 块中提取 plan 字段。第四层，planContent 字段，从 user message 的 planContent 字段提取。第五层，plan_file_reference，从 auto-compact 创建的附件中提取。

为了应对远程场景，Claude Code 会在两个时机调用 persistFileSnapshotIfRemote，把计划内容作为 file_snapshot 系统消息写入 transcript。第一个时机是 normalizeToolInput 处理 ExitPlanMode 工具时，也就是 switch 语句中的 EXIT_PLAN_MODE_V2_TOOL_NAME 分支，位于 src/utils/api.ts 第 578 行。第二个时机是 ExitPlanMode 的 call 方法写入用户编辑后的计划时，位于 ExitPlanModeV2Tool.ts 第 260 行。这是远程会话中唯一可靠的持久化渠道。

Fork 会话用 copyPlanForFork 函数处理，位于 src/utils/plans.ts 第 239 到 264 行，逻辑更简单，但有一个关键细节：要生成新的 slug。如果复用原始 slug，原始会话和 fork 会话会写入同一个文件，导致互相覆盖。

9.6 退出 Plan 模式：审批与权限恢复

退出是整个 Plan 模式中最复杂的部分，因为它需要同时处理权限恢复、用户审批、计划同步和多种执行上下文。

9.6.1 验证：必须在 Plan 模式中

ExitPlanModeV2Tool 的第一道检查位于 src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 195 到 219 行。（代码从略：这段代码是 validateInput 方法。如果调用者是 teammate，直接放行。否则读取当前模式；如果不在 plan 模式，就记录一条名为 tengu_exit_plan_mode_called_outside_plan 的事件，并返回校验失败、一条提示消息和错误码 1。在 plan 模式中则返回通过。）

为什么还需要这个检查？因为模型有时会“忘记”自己已经退出了 Plan 模式，然后再次调用 ExitPlanMode。这种“失忆”主要有三个原因。第一，上下文压缩，即 Compact，清除了关键信号：早期的 ExitPlanMode 成功消息在压缩后可能被丢弃，模型看不到已经退出的证据。第二，Deferred tool 列表的误导：模型在工具列表中仍然看到 ExitPlanMode 可用，容易误认为自己还在 Plan 模式中。第三，模式状态缺乏显式标记：当前模式信息主要通过系统附件传递，如果附件因节流未注入，模型可能产生状态混淆。

这个检查避免了重复退出导致的状态混乱。

9.6.2 用户审批

通过 checkPermissions 方法触发权限请求对话框，位于 src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 221 到 239 行。（代码从略：这段代码在调用者是 teammate 时直接返回允许；否则返回一个 ask 行为，消息是 Exit plan mode，也就是询问用户是否退出计划模式，并把输入原样带回。）

用户在审批对话框中有三种选择。第一是直接批准，计划原样通过。第二是编辑后批准，修改计划内容后再批准，修改通过 permissionResult 里的 updatedInput 中的 plan 字段传回。第三是拒绝，不退出 Plan 模式，继续修改计划。

9.6.3 权限恢复的精密逻辑

call 方法中的权限恢复是最精妙的部分，位于 src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 357 到 403 行。（代码从略：这段逻辑在应用状态更新时先判断，如果当前模式不是 plan 就原样返回；否则把已退出 Plan 模式的标志和需要注入退出附件的标志都置为真。接着确定要恢复的模式：优先取进入 Plan 模式前记住的 prePlanMode，没有就用 default。这里有一个断路器防御：如果启用了 TRANSCRIPT_CLASSIFIER 特性，而恢复模式是 auto，但 auto gate 已经关闭，就安全回退到 default，不能绕过断路器；同时按恢复结果设置 auto 模式是否活跃。然后做权限规则恢复：如果恢复到 auto，就保持危险权限被剥离；如果恢复到非 auto 且之前剥离过危险权限，就把它们还原。最后返回新状态，把模式设为恢复出的模式，并把 prePlanMode 清空，防止下次退出时误用。）

这段逻辑处理了一个非常细腻的边缘情况：断路器防御。假设用户原来在 auto 模式，进入 Plan 模式时系统记住了 prePlanMode 为 auto。但在 Plan 模式期间，auto 模式的断路器触发了，比如连续失败次数超过阈值，auto gate 被关闭。此时如果盲目恢复到 auto，就相当于绕过了断路器——这是不允许的。所以要先检查 isAutoModeGateEnabled，如果 gate 已关闭，就回退到 default。

9.6.4 四种结果消息

mapToolResultToToolResultBlockParam 函数根据不同的上下文返回 4 种不同的消息，位于 src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 419 到 492 行。第一种上下文是 teammate 等待审批，返回的消息是计划已提交给团队负责人，并附上 Request ID，后续模型等待 inbox 消息。第二种上下文是子 Agent，返回的消息是用户已批准计划，请回复 ok，子 Agent 随之结束。第三种上下文是空计划，返回的消息是用户已批准退出计划模式，可以继续，模型直接开始工作。第四种是正常审批，返回的消息是用户已批准你的计划，并附上完整计划文本，模型按计划实施。

正常审批的消息里会回传完整的计划文本。这不是冗余——它确保模型在后续实施中可以直接引用计划内容，不必重新读取计划文件。如果用户编辑了计划，标签会变成 Approved Plan (edited by user)，提醒模型注意用户的修改。

9.7 状态管理的全局视图

Plan 模式的状态分散在多个位置，但通过 handlePlanModeTransition 函数统一管理转换逻辑，位于 src/bootstrap/state.ts 第 1349 到 1363 行。（代码从略：这段代码根据转换方向设置标志。从其他模式进入 plan 时，清除遗留的退出附件标志。从 plan 退出到其他模式时，触发一次性退出附件。）

完整的状态字段如下。（代码从略：这段代码列出了全局状态中与 plan 相关的字段。hasExitedPlanMode 是会话级标志，表示是否曾退出 plan 模式。needsPlanModeExitAttachment 是一次性标志，表示下一轮注入退出消息。needsAutoModeExitAttachment 也是一次性标志，对应 auto 模式的退出消息。planSlugCache 是从会话 ID 到 slug 的映射。此外，ToolPermissionContext 中与 plan 相关的字段有三个。mode，取值是 default、plan、auto 或 bypassPermissions。prePlanMode，记录进入 plan 前的模式，取值是 default、auto 或 bypassPermissions。strippedDangerousRules，保存 auto 模式被剥离的危险权限，在 plan 期间保存。）

needsPlanModeExitAttachment 和 needsAutoModeExitAttachment 是一次性标志，英文叫 fire-once flag。它们在被消费后立即清除，确保退出消息只注入一次。这种设计避免了“退出 plan 模式”的通知在后续每轮都重复出现。

9.8 重入 Plan 模式

如果用户在同一会话中第二次进入 Plan 模式，系统会注入一条特殊的重入指引，位于 src/utils/messages.ts 第 3829 到 3847 行。指引说：你此前退出过 plan 模式，现在是重新返回；上一次规划会话在 planFilePath 指向的路径留有一个计划文件。在开始任何新规划之前，应该这样做。第一，读取现有的计划文件，了解之前规划了什么。第二，用用户当前的请求对照那份计划。第三，决定如何继续：如果是不同的任务，就覆盖旧计划、重新开始；如果是同一任务的延续，就在现有计划上修改。第四，在调用 ExitPlanMode 之前，一定要先编辑计划文件。

这条指引解决了一个实际问题：模型可能误以为旧计划仍然有效。指引明确要求“先读取旧计划、再判断是否相关”，避免在过期的计划上继续工作。

9.9 与其他系统的交互

9.9.1 与权限系统

Plan 模式深度集成在权限系统中，详见第 12 章。prepareContextForPlanMode 的行为取决于进入前的模式。从 default 进入 plan 时，简单保存 prePlanMode 为 default。从 auto 进入 plan，且 shouldPlanUseAutoMode 为 false 时，先关闭 auto mode，再恢复被剥离的危险权限，然后把 needsAutoModeExitAttachment 置为真，最后保存 prePlanMode 为 auto。从 auto 进入 plan，且 shouldPlanUseAutoMode 为 true 时，保持 auto mode 活跃，并保存 prePlanMode 为 auto。

9.9.2 与多 Agent 系统

Plan 模式为多 Agent 架构中的子 Agent 提供专用指令，多 Agent 架构详见第 8 章，相关指令位于 src/utils/messages.ts 第 3399 到 3417 行。子 Agent 的计划文件使用独立的命名空间，也就是在主 slug 后面加上 agent 和 agent 编号，避免与主会话的计划文件冲突。

9.9.3 与 normalizeToolInput

src/utils/api.ts 第 572 到 580 行中，normalizeToolInput 对 ExitPlanMode 做了特殊处理——从磁盘读取计划内容并注入到工具输入中。（代码从略：这段代码在遇到 EXIT_PLAN_MODE_V2_TOOL_NAME 时，先取出计划内容和计划文件路径，并调用 persistFileSnapshotIfRemote 做远程持久化；如果计划内容存在，就把它和计划文件路径合并进工具输入再返回。）

注入的 plan 和 planFilePath 字段供 Hook 和 SDK 消费者使用，但在发送给 API 之前会被 normalizeToolInputForAPI 剥离——因为 API schema 中没有这些字段。

9.10 设计洞察

这一章有五条设计洞察。

第一条，主动降权换信任。Plan 模式是整个 Claude Code 中唯一一个“模型主动要求降低自己权限”的机制。这种设计把“我需要权限”的传统模式，反转成“我主动放弃权限以换取你的信任”。当模型判断任务复杂时，它选择先束缚自己的双手，只用眼睛看，直到你说“可以动手了”。

第二条，对称的状态转换。进入时把当前模式记入 prePlanMode，退出时把 prePlanMode 还给当前模式。这种对称性让 Plan 模式在大多数情况下能干净地恢复到进入前的模式。但它并非严格的“纯函数”：退出时会置若干一次性退出标志，比如 setHasExitedPlanMode 和 setNeedsPlanModeExitAttachment 等；而且断路器触发时，auto 会被降级为 default，详见 9.6.3 节。此时退出后的模式与进入前并不完全一致。

第三条，渐进式 prompt 注入。full 到 sparse 再到 full 的节流策略，在 token 成本和模型记忆之间找到了平衡。完整指令约 4700 字符，约合 1200 token；sparse 提醒仅 300 字符，约合 75 token。按平均 15 轮的 plan 会话计算，节流策略节省了约 10000 token。

第四条，实验驱动的迭代。Phase 4 的四种变体不是拍脑袋设计的。它们基于 2630 万条基线样本，也就是 N 等于 26.3M，时间跨度 14 天，按 plan-exit 计。团队用科学的 A/B 测试验证“更短的计划是否导致更好的用户满意度”。这体现了工程团队的优化思路：看重用户满意度，也就是拒绝率，而非技术指标，比如计划长度。

第五条，容灾设计无处不在。从 5 层计划恢复策略、到断路器防御、到重入引导，Plan 模式的每个环节都在问：如果出了问题怎么办？这不是过度设计——在一个 AI 系统里，模型的行为天然难以预测，防御性编程是唯一合理的策略。

动手实践：尝试在 Claude Code 中输入 /plan 重构你项目中最复杂的模块，观察模型如何探索代码、生成计划、等待你的审批。然后编辑计划文件，也就是 ~/.claude/plans/ 目录下的那些 md 文件，再批准，观察模型如何处理你的修改。

上一章是多 Agent 架构，下一章是代码编辑策略。
