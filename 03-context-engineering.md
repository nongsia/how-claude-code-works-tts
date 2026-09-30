---
title: 第 3 章：上下文工程（朗读版）
---

第 3 章：上下文工程

本文是 docs/03-context-engineering.md 的朗读版，表格与代码已转为口语描述，内容未增删。

上下文工程是 Claude Code 能力的隐形支柱。模型的决策质量完全取决于它看到了什么上下文。

为什么上下文工程如此重要？

LLM 有一个固定大小的上下文窗口，Claude 当前最大 1M token。而一次真实的编码会话，可能涉及几十次文件读取、数百次工具调用，产生的原始文本量轻松超过百万 token，很容易逼近甚至超出上下文窗口的容量。

这意味着系统必须做出艰难的取舍：哪些信息留在上下文中，哪些被压缩或丢弃。如果取舍不当，模型会忘记刚才编辑了哪个文件、重复读取已经看过的内容、或者产生与之前决策矛盾的输出。

可以把上下文窗口想象成一张办公桌：桌面有限，你必须把最重要的文档放在手边，其他的归档到抽屉里。上下文工程就是这套"文档管理系统"——决定桌上放什么，也就是上下文构建；什么时候把旧文档收进抽屉，也就是压缩；以及如何让归档的文档在需要时快速取回，也就是持久化与恢复。

但上下文工程面临的挑战不止于此。Claude Code 的每次 API 请求，光系统提示词和工具定义就可能有 50 到 100K token。为了避免每次都从零处理这些内容，Claude Code 依赖服务端的前缀缓存，英文叫 Prefix Caching，也叫 KV Cache。服务端记住之前处理过的前缀，后续请求只处理新增部分，省去重复消化整段前缀的延迟和成本。

但前缀缓存有一个残酷的约束：前缀必须字节级完全一致才能命中缓存。不是"差不多就行"，而是任何一个字节的变化，哪怕只是换了一个请求头、改了一个工具的顺序，都会导致整个前缀的缓存失效，50 到 100K token 全部需要重新处理。

这给上下文工程带来了一种"带着镣铐跳舞"的感觉：你不能随意调整提示词顺序，不能随意增删工具定义，不能中途改变请求元数据。每一个设计决策都必须同时满足两个目标——给模型最好的上下文，同时不打破缓存。本章中你会反复看到这种张力：很多看起来"过度设计"的机制，背后的驱动力都是缓存稳定性。

Claude Code 在这方面的工程量远超大多数人的预期。本章将深入分析它的完整上下文管理体系。

本章关键文件：src/context.ts，共 190 行；src/utils/api.ts；以及 src/services/compact 目录。

3.1 上下文构建全景

每次调用 Claude API，模型都是从零开始的——它没有跨请求的持久记忆，只能看到当前请求中携带的内容。因此，Claude Code 必须在每次 API 调用前，将模型需要的所有信息组装成一个完整的请求。

这个组装过程涉及三大支柱：

第一，系统提示词，英文叫 System Prompt。它定义模型的身份、能力边界和行为规则，是最稳定的部分，跨请求基本不变。

第二，系统和用户上下文，即 System 与 User Context。包括环境信息，比如 git 状态和平台，以及项目知识，也就是 CLAUDE.md 指令文件。每会话计算一次。

第三，消息历史，即 Message History。包括用户的提问、模型的回答、工具调用和结果，记录了对话中发生的一切。这是变化最快、占用空间最大的部分。

用一张结构图来表示三大支柱如何组装。首先，系统提示词由五个来源组装而成，分别是归属头、CLI 系统提示词前缀、工具描述与 prompt、工具搜索指令、顾问指令，它们共同汇成完整的系统提示词。其次，系统上下文由 getSystemContext 提供，内容是 Git 状态，包括分支、暂存区和最近提交。再次，用户上下文由 getUserContext 提供，包括 CLAUDE.md 文件的发现结果和 ISO 格式的当前日期。最后，完整系统提示词、systemContext、userContext，再加上对话历史 messages，四者一起构成最终的 API 请求。

一次 API 请求的完整解剖

上面的三大支柱比较抽象，一次真实的 API 请求到底长什么样？Claude API 的请求体有三个顶级字段：system，是系统提示词数组；tools，是工具 schema 数组；messages，是消息数组。

一张结构解剖图展示了完整布局。第一块是 system 数组，由多个 TextBlock 拼接：第 0 项是归属头，即 Attribution Header，不缓存；第 1 项是 CLI 前缀，即交互模式或 -p 模式的指令，也不缓存；接下来进入静态内容区，第 2 项是核心指令、工具描述、安全规则和行为准则，对所有用户完全相同，这一段标记为 global 缓存；然后是哨兵标记 SYSTEM_PROMPT_DYNAMIC_BOUNDARY，也就是动态边界；边界之后是动态内容区，第 3 项是输出风格、语言偏好、MCP 指令等，因用户和会话而异，不缓存。

第二块是 tools 数组：先是内置工具，如 Read、Edit、Bash、Grep、Write、Glob 等；然后是用户安装的 MCP 工具，可能标记 defer_loading 延迟加载；advisor 等服务端工具排在数组末尾。整个数组渲染在 system 之前，随 system 断点一并缓存；快照中工具没有单独打 cache_control，这一点见 3.6 节。

第三块是 messages 数组：第一条用户消息是 system-reminder，包含 CLAUDE.md 内容和当前日期，会话开始时计算一次，标记为 isMeta；接着是用户第 1 条消息、模型回复，可能包含 tool_use 块，以及 tool_result 结果；再往后是若干附件消息，每条都是独立的 isMeta 用户消息，分别包裹着记忆文件内容、可用技能列表、延迟工具发现结果、MCP 指令增量的 system-reminder；然后是模型第 2 轮回复。消息不断增长，直到压缩机制介入。

注意几个关键设计：

第一，记忆、技能、MCP 指令不在系统提示词中，而是作为 system-reminder 附件消息注入到消息数组中。这样做的好处是：它们可以按需、增量地注入，只在内容变化时添加新的附件，而不会破坏系统提示词的缓存。

第二，CLAUDE.md 和日期虽然是"元信息"，但被放在消息数组的第一条，而非系统提示词中。因为 CLAUDE.md 内容因项目而异，放在系统提示词中会降低缓存共享率。

第三，工具 schema 数组整体渲染在 system 之前，被 system 静态块上的断点一并缓存。快照里保留了给工具单独打 cache_control 的能力，但主查询路径没有启用，见 3.6 节。

各组件在一个会话中的变化特征，可以总结成八句话。第一，核心系统指令，位于 system 数组中静态边界之前，会话内从不变化，所有用户、所有会话完全相同，属于全球共享的全局缓存。第二，动态系统指令，位于静态边界之后，会话内也从不变化，它因用户而异但在会话内固定，会话开始时确定。第三，工具 schema，位于 tools 数组，变化极少，只在 MCP 重连或 Tool Search 发现新工具时变化，延迟加载减少了变动。第四，CLAUDE.md 加当前日期，位于 messages 的第 0 条，从不变化，会话开始时 memoize 计算一次，包裹在 system-reminder 中。第五，用户消息和模型回复，位于 messages，每轮增长，由压缩机制控制增速。第六，工具调用和结果，位于 messages，每次工具执行后增长，旧结果可被 Microcompact 清理。第七，记忆文件，以附件形式位于 messages，按需注入，相关记忆被发现时注入并去重，每会话最多 60KB。第八，技能和 MCP 指令，同样以附件形式位于 messages，增量注入，只在列表变化时注入增量，不重复发送已知内容。

一轮对话如何改变上下文

看完上面的静态结构，再看动态过程——一轮对话，即 Turn N 里，上下文经历了哪些变化：

流程图描述如下。首先，Turn N 开始，messages 中是历史消息，先做压缩检查，走五级流水线，依次是 Tool Result 预算裁剪、History Snip、Microcompact、Context Collapse、Autocompact。然后进入第二步，组装请求，把 system 和 tools 准备好，并对 messages 前置用户上下文，接着发送 API 请求，流式接收 assistant 消息，其中可能包含 tool_use。此时判断有没有 tool_use：没有，Turn N 就结束，等待用户输入；有，就执行工具并收集 tool_result，再收集附件消息，包括记忆预取、技能增量、MCP 指令增量等，包装为 system-reminder，最后把 assistant 消息、tool_result 和附件拼接到消息数组，进入 Turn N+1 的工具循环。

核心要点：上下文是"活的"——每一轮对话，消息数组都在增长，即新的 assistant 回复、tool_result 加附件，而压缩机制在每轮开始时检查并控制增长速度。系统提示词和工具列表在整个会话中基本不变，这正是它们能被高效缓存的原因。

3.2 系统提示词的构建

系统提示词是上下文中最稳定的部分——它定义了模型"是谁"以及"该怎么做"。正因为稳定，它也是提示词缓存的最佳候选。Claude Code 的系统提示词构建在稳定性和灵活性之间做了精心平衡。

归属头，即 Attribution Header

基于指纹的身份标识，用于追踪请求来源。

CLI 系统提示词前缀

根据运行模式变化：交互式模式，即 REPL，和 -p 单次查询模式有不同的前缀指令。

系统提示词优先级

系统提示词的构建有严格的优先级，由 buildEffectiveSystemPrompt 函数实现，位于 src/utils/systemPrompt.ts。

（代码从略：这段注释列出了从高到低的优先级。第 0 级是 overrideSystemPrompt，完全覆盖，例如 loop 模式；第 1 级是 coordinatorSystemPrompt，协调器模式，受 feature 开关控制；第 2 级是 agentSystemPrompt，即 Agent 定义的提示词，Proactive 模式下追加到默认提示词后面，普通模式下替换默认提示词；第 3 级是 customSystemPrompt，由 --system-prompt 参数指定；第 4 级是 defaultSystemPrompt，即标准 Claude Code 提示词。另外 appendSystemPrompt 始终追加到末尾，override 模式除外。）

这个优先级链确保了不同运行模式，包括交互、Agent、协调器、SDK，都能获得正确的系统提示词，同时保留用户自定义的能力。

静态与动态边界标记

系统提示词中有一个关键的设计元素——SYSTEM_PROMPT_DYNAMIC_BOUNDARY，位于 src/constants/prompts.ts 第 114 行。这是一个哨兵字符串 __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__，它将系统提示词数组分成两半：

边界之前：核心指令、工具描述、安全规则等——对所有用户的所有会话都完全相同的内容。

边界之后：MCP 工具指令、输出风格、语言偏好等——因用户或会话而异的内容。

为什么需要这个边界？因为它直接影响提示词缓存的效率。边界之前的静态部分可以使用全局档缓存，即 scope 为 global，跨所有用户共享——这意味着全球数百万 Claude Code 用户可以共享同一份缓存的核心系统提示词。边界之后的动态部分则只能用 org 档缓存，或者不缓存。没有这个边界，整个系统提示词都只能做 org 级别缓存，浪费大量缓存存储在完全相同的内容上。

Section 级缓存

系统提示词的各个组成部分通过 systemPromptSections.ts 实现了 section 级别的缓存。这里有两种类型：

（代码从略：第一种是 systemPromptSection，计算一次就缓存，直到 clear 或 compact 命令才失效，例如工具指令的构建。第二种是 DANGEROUS_uncachedSystemPromptSection，每轮重新计算，会破坏提示词缓存，例如 MCP 指令部分，因为 MCP server 可能在轮次之间连接或断开；调用时必须提供一个理由参数，说明为什么缓存破坏是必要的。）

DANGEROUS 前缀是刻意为之的代码级警示——它提醒开发者：这个 section 每轮都会重新计算，如果值发生变化会破坏提示词缓存。开发者必须提供一个 reason 参数解释为什么缓存破坏是必要的。大多数 section 都是稳定的，比如工具描述、安全规则；只有少数依赖轮次间可能变化的运行时状态的 section，例如 MCP server 可能在轮次之间连接或断开，需要使用 DANGEROUS 变体。

clearSystemPromptSections 在 clear 和 compact 命令时调用，同时重置 beta header 锁存，详见 3.6 节第二层防御，让下一次对话获得完全新鲜的状态。

系统上下文，即 getSystemContext

来自 src/context.ts 的 getSystemContext 函数，被 memoize 缓存，每会话只计算一次。

完整的实现展示了一个精心设计的上下文收集过程：

（代码从略：这是 src/context.ts 中的 getGitStatus 函数，经 memoize 缓存。它先判断当前目录是否是 git 仓库，不是就返回空。然后并行执行 5 个 git 命令，分别获取当前分支、默认主分支、简短状态、最近 5 条提交记录和 git 用户名。状态如果超过 2000 字符就会被截断，防止大量未提交文件撑爆上下文。最后把各段信息拼接返回，其中第一条至关重要，是一句免责声明，告诉模型这是会话开始时的 git 状态快照，不会在对话过程中更新，防止模型在后续轮次中幻觉出实时的 git 状态。出错时记录错误并返回空。）

值得注意的设计细节：

第一，Promise.all 并行：5 个 git 命令同时执行，而不是串行等待——这在大型仓库中可以节省数百毫秒。

第二，--no-optional-locks 参数：避免 git 命令获取锁，导致与其他 git 操作冲突。

第三，MAX_STATUS_CHARS 等于 2000：限制状态输出长度。想象一个有 500 个未提交文件的 monorepo——不截断的话，git status 本身就会消耗大量上下文预算。

第四，Disclaimer 文本：明确告诉模型这是快照，不会实时更新——这是防止模型幻觉的重要手段。

getSystemContext 本身还有条件跳过逻辑：

（代码从略：getSystemContext 同样被 memoize 缓存。在 CCR，也就是 Cloud Code Remote 模式下，或者禁用 git 指令时，跳过 git 状态收集。另外还有一个内部调试用的缓存断裂注入，受 feature 开关控制，启用时在返回值中附带 cacheBreaker 字段。）

用户上下文，即 getUserContext

（代码从略：getUserContext 同样被 memoize 缓存。它处理了 bare 模式的微妙语义：环境变量 CLAUDE_CODE_DISABLE_CLAUDE_MDS 是硬关闭，始终生效；而 bare 模式只是跳过自动发现，也就是遍历当前工作目录，但尊重显式指定的附加目录。代码注释原文解释了这层语义：bare 意味着跳过我没有主动要求的，而不是忽略我明确要求的。接下来获取 CLAUDE.md 内容，并把结果缓存起来供 yoloClassifier 使用，避免创建 import 循环。最后返回 CLAUDE.md 内容和当前日期。）

CLAUDE.md 发现机制

CLAUDE.md 是 Claude Code 的项目级指令文件，类似于 .editorconfig 或 .eslintrc，但面向 AI Agent。它的发现过程比看起来要复杂得多。

发现顺序由 getMemoryFiles 实现，分五步：

第一步，管理策略文件：从 MDM，即移动设备管理策略中读取的指令，例如 /etc/claude-code/CLAUDE.md。

第二步，用户主目录：~/.claude/CLAUDE.md 下的全局配置。

第三步，项目文件：从 CWD 向上遍历目录树，查找每一层的指令文件。

第四步，本地文件：CLAUDE.local.md，不提交到 git 的个人指令。

第五步，显式附加目录：--add-dir 参数指定的额外目录。

文件名模式：每个目录下检查 CLAUDE.md、.claude/CLAUDE.md，以及 .claude/rules/ 目录下的所有 .md 文件。这意味着你可以将不同领域的指令拆分成独立文件，例如 .claude/rules/testing.md、.claude/rules/style.md，系统会自动加载它们。

优先级排序：文件按从远到近的顺序加载——靠近 CWD 的文件后加载，因此优先级更高。这符合"就近原则"：项目根目录的全局规则可以被子目录的局部规则覆盖。由于 LLM 对上下文末尾的内容关注度更高，也就是近因效应，后加载的指令在模型的"注意力"中权重更大。

include 指令，位于 src/utils/claudemd.ts：

CLAUDE.md 文件可以通过 @ 语法引用其他文件。

（代码从略：示例展示了一个项目指令文件，标题是"项目指令"，正文用三行 @ 语法分别引用相对路径 docs 目录下的 coding-standards.md、用户主目录下的 global-rules.md，以及绝对路径 /etc/company-policy.md。）

引用规则有七条。@ 加路径、不带前缀时，等同于显式写出 ./ 的相对路径形式，按相对路径解析。@ 加波浪线加路径，从用户主目录解析。@ 加斜杠加路径，按绝对路径解析。引用只在叶子文本节点中生效，代码块内的 @ 符号不会被解析。被引用的文件作为独立条目插入到引用文件之前。系统通过跟踪已处理文件防止循环引用。只允许文本文件扩展名，例如 .md、.txt 等，防止加载二进制文件。

过滤：filterInjectedMemoryFiles 在 feature flag，即 tengu_moth_copse，开启时，剔除 AutoMem 或 TeamMem 类型的记忆条目——这些自动记忆改由附件消息按需注入，不再进入 CLAUDE.md 上下文块。

缓存失效：clearMemoryFileCaches 纯清缓存、不触发 hook，用于 worktree 切换和 settings 同步时；resetGetMemoryFilesCache 在压缩或 clear 命令时调用，使下一次 getMemoryFiles 重新加载，并以相应原因触发 InstructionsLoaded hook。

上下文注入顺序

（代码从略：src/utils/api.ts 中，完整系统提示词的构造方式是，把系统上下文追加到系统提示词之后，再转成系统提示词数组，也就是系统上下文后置；而用户上下文则在消息前置。）

系统上下文后置于系统提示词，用户上下文前置于消息——这个顺序影响提示词缓存的效率。系统提示词是最稳定的部分，跨请求不变，放在最前面有利于缓存命中；而用户上下文，即 CLAUDE.md 和日期，可能随会话变化，放在消息前面不会破坏系统提示词的缓存。

3.3 消息历史管理

Claude Code 不是简单地将所有历史消息发送给 API。它通过一系列机制管理消息列表，确保发送给 API 的消息格式合法、内容精简。

压缩边界，即 Compact Boundary

当 autocompact 发生后，消息列表中会插入一个 compact_boundary 标记。之后的 API 调用只发送边界之后的消息：

（代码从略：只有一行，把 messages 传入 getMessagesAfterCompactBoundary，取压缩边界之后的消息，展开成本次查询的消息列表。）

当 HISTORY_SNIP 功能启用时，还会在此基础上投影一个"剪裁视图"——将被标记为 snipped 的消息从 API 请求中隐藏。

消息规范化，即 normalizeMessagesForAPI

normalizeMessagesForAPI 位于 src/utils/messages.ts，约 380 行，是消息发送前的关键处理步骤。它解决了一个核心问题：Claude Code 内部的消息格式和 API 要求的消息格式不完全一致。

（代码从略：这个函数先建立可用工具名的集合，然后分步处理。第一步，附件重排序，并过滤虚拟消息。第二步，构建错误到块类型的映射，例如 PDF 太大、图片太大，并记录每条消息需要剥离的块类型。第三到第七步遍历消息逐条处理：剥离 tool_reference、advisor 块和错误的媒体项；处理 thinking 和 signature 块；合并同 ID 的分裂 AssistantMessage；验证和修复 tool_use 与 tool_result 的配对。）

下面是每个处理步骤及其解决的问题：

第一步，附件重排序，即 reorderAttachmentsForAPI：附件消息在内部可能出现在任意位置，但 API 要求它们在语义上关联的消息之前。此步骤将附件消息向上冒泡，直到遇到 tool_result 或 assistant 消息为止。如果不做这一步，API 可能看到一个孤立的图片块，却不知道它与哪条消息相关。

第二步，过滤虚拟消息：标记为 isVirtual 的消息被移除，例如 REPL 内部工具调用的临时消息。这些消息的存在仅为了 UI 展示——例如自动触发的内部操作在界面上需要显示进度，但它们不应进入 API 请求。

第三步，构建错误到块类型的映射：某些 API 错误，如"PDF 太大"、"图片太大"，需要从后续消息中剥离对应的媒体块。系统构建一个映射表 errorToBlockTypes，将错误文本映射到需要剥离的块类型，即 document 和 image。如果不做这一步，同一个过大的 PDF 会在每次请求中被发送，每次都触发同样的错误。

第四步，剥离内部元素：从消息中移除 tool_reference，即工具引用标记；移除 advisor blocks，即顾问指令；移除因 API 错误而需要剥离的媒体项。tool_reference 是延迟工具加载系统，也就是 Tool Search 的内部跟踪标记，API 对此毫无概念——它们的存在会导致 API 返回格式错误。

第五步，thinking 和 signature 块处理：根据模型要求处理思考块。某些模型不支持 thinking 或 redacted_thinking 块——发送它们会直接导致 API 返回 400 错误。Signature 块用于验证思考块的完整性，也需要在不支持的模型上剥离。

第六步，合并分裂消息：流式解析器可能将一个 API 响应拆分为多条具有相同 message.id 的 AssistantMessage，这发生在并行工具调用产生多个 content block 的时候。API 期望一个响应对应一条消息，多条同 ID 消息会违反消息交替规则。

第七步，验证和修复配对：API 要求每个 tool_use 块都有对应的 tool_result，反之亦然。会话崩溃、压缩、中途中断都可能破坏这种配对关系。此步骤检测并修复孤儿块——为缺失结果的 tool_use 生成错误类型的 tool_result，为缺失请求的 tool_result 注入合成的 tool_use。没有这一步，恢复一个崩溃的会话几乎必然会因为配对不完整而报错。

为什么这么复杂？因为 Claude API 对消息格式有严格要求：用户和助手消息必须交替出现，tool_use 和 tool_result 必须配对，thinking 块不能出现在不支持的位置。而 Claude Code 的内部消息列表可能因为会话崩溃恢复、压缩操作、用户中断等原因违反这些约束。normalizeMessagesForAPI 是防御层——确保无论内部状态多混乱，API 始终收到合法的消息序列。

3.4 五级压缩流水线

这是 Claude Code 上下文管理的核心机制。当对话越来越长，Token 使用量不断增长，五级压缩流水线逐级启动。设计哲学是渐进式压缩——先用成本最低的手段尝试释放空间，只在必要时才动用更重的武器。

流程图描述如下。消息列表从第一级开始：第一级是 Tool Result 预算裁剪，由 applyToolResultBudget 执行，把大结果持久化到磁盘；第二级是 History Snip 剪裁，由 snipCompactIfNeeded 执行，是受 feature 开关控制的释放 Token 手段；第三级是 Microcompact 微压缩，有基于时间和缓存编辑两条路径，清理旧工具结果；第四级是 Context Collapse 上下文折叠，采用投影式只读视图，不修改原始消息；第五级是 Autocompact 自动全量压缩，fork 子 Agent 生成摘要，是最后手段。

为什么按此顺序执行？

每一级都比前一级"更重"——消耗更多计算资源，或丢失更多上下文细节：

第一，Tool Result Budget 最先：纯本地操作，不调用 API。大结果写入磁盘，上下文只保留预览。零延迟、零成本。

第二，Snip 释放最多：直接从消息列表中移除冗余部分，释放大量 Token，可能使后续压缩不必要。

第三，Microcompact 成本极低：清理旧工具结果，不调用 API，适合频繁执行。

第四，Context Collapse 在 Autocompact 之前：Context Collapse，即上下文折叠，是一种投影式压缩——创建消息的只读折叠视图，不修改原始数据，详见 Level 4 一节。折叠可能使 Token 使用量降到 Autocompact 阈值以下，从而阻止不必要的全量压缩——保留了更细粒度的上下文。

第五，Autocompact 作为最后手段：需要 fork 一个子 Agent 调用 API 生成摘要，成本最高，且不可逆，原始消息被摘要替换。

各级压缩详解

Level 1：Tool Result 预算裁剪

applyToolResultBudget 是最轻量的处理——纯本地操作，不调用 API。它解决的核心问题是：单次工具调用可能返回巨大的结果。例如，用 FileReadTool 读取一个万行文件，或用 BashTool 执行 find 命令获取数千个文件路径。

处理机制位于 src/utils/toolResultStorage.ts，分三点：

第一，每个工具声明一个 maxResultSizeChars，默认值为 50000 字符，即 DEFAULT_MAX_RESULT_SIZE_CHARS。

第二，可通过 GrowthBook Feature Flag，即 tengu_satin_quoll，按工具名覆盖阈值。

第三，当工具结果超过阈值时，不是简单截断，而是持久化到磁盘。

持久化路径是：项目目录下的会话 ID 目录中的 tool-results 目录，文件名是 tool_use_id，扩展名是 txt 或 json。

上下文中只保留一个紧凑的引用消息：

（代码从略：这是一段引用消息模板。它告诉模型输出太大，本例是 2.3 MB；完整输出已保存到某个会话目录下 tool-results 路径中的 txt 文件；然后附上前 2KB 的内容预览，即前 2000 字节的预览。）

为什么选择持久化而非截断？截断意味着数据永久丢失——如果模型后来需要查看完整输出，比如在第 500 行发现了 bug，它无法恢复。持久化则保留了完整数据，模型可以随时使用 Read 工具读取磁盘文件来获取完整内容。2KB 的预览给了模型足够的信息来判断是否需要查看完整结果。

此外，applyToolResultBudget 还会追踪已替换的工具结果，即 ContentReplacementState，确保会话恢复时做出与原始会话完全相同的替换决策，维持提示词缓存的稳定性。

Level 2：History Snip

snipCompactIfNeeded 是受 HISTORY_SNIP feature 开关控制的功能，通过剪裁历史消息中的冗余部分释放 Token。释放量通过 snipTokensFreed 传递给后续的 autocompact 阈值检查。这很重要，因为 snip 移除了消息，但最后一条 assistant 消息的 usage 仍然反映 snip 前的上下文大小，不做修正会导致 autocompact 过早触发。

Level 3：Microcompact

Microcompact 是 Claude Code 压缩体系中最精巧的一环。它的目标是清理历史中不再需要的旧工具结果——如果你 30 分钟前读取了一个文件，那个工具结果大概率已经不再有用，但它可能还占着数千 Token。

关键设计：Microcompact 有两条完全不同的路径，根据缓存状态选择：

路径 A：基于时间的 Microcompact，缓存已冷。

当用户离开一段时间后回来，即距上次 assistant 消息超过配置的分钟数，服务端的提示词缓存已经过期，默认 5 分钟 TTL。此时缓存已经"冷了"，无论怎么做都需要重新上传完整前缀。

在这种情况下，Microcompact 直接修改消息内容：

（代码从略：一行代码，把旧工具结果的内容替换为占位符，占位文本是：旧工具结果内容已清除。）

只保留最近 N 个可压缩工具的结果，即 keepRecent，最少保留 1 个，其他全部替换为占位符。因为缓存已经冷了，修改消息内容不会造成额外的缓存失效——缓存本来就需要重建。

可压缩的工具类型：FileRead、Shell 或 Bash、Grep、Glob、WebSearch、WebFetch、FileEdit、FileWrite。

路径 B：缓存编辑 Microcompact，缓存仍热。

当缓存还没过期时，即用户一直在活跃对话，情况完全不同。如果直接修改消息内容，会导致缓存键变化，使 100K 以上的 token 缓存前缀全部失效，需要重新上传和计费。

因此，缓存编辑路径完全不修改本地消息。它使用一种巧妙的 API 级机制：

第一步，在工具结果块上添加 cache_reference 字段，值等于 tool_use_id，让服务端能够定位缓存中的具体位置。

第二步，构造 cache_edits 块，告诉服务端删除这些 cache_reference 指向的内容。

第三步，服务端在缓存中就地删除，不需要客户端重新上传前缀。

（代码从略：两行注释加一个返回语句。要点是：消息本身不变，编辑在 API 层发生，cache_edits 块通过 consumePendingCacheEdits 传递给 API 层，函数把 messages 原样返回。）

已发出的 cache_edits 通过 pinCacheEdits 保存，在后续请求中按原始位置重新发送，因为服务端需要看到它们才能维持缓存一致性。

两条路径的对比：触发条件上，基于时间的 Microcompact 是时间间隔超过阈值、缓存已冷；缓存编辑 Microcompact 是工具数量超过阈值、缓存仍热。操作方式上，前者直接修改消息内容，后者使用 cache_edits API 块。对缓存的影响上，前者因为缓存本来就要重建而无额外影响，后者保持缓存热度、避免重新上传。API 调用上，两者都是零次，后者的编辑在下次正常请求中捎带。适用场景上，前者适合用户回来后的首次请求，后者适合活跃对话中的持续清理。

两条路径互斥：时间触发优先级更高，如果时间触发生效，会跳过缓存编辑路径，因为缓存已冷，使用 cache_edits 没有意义。

路径 C：API 端原生 context management

除了上面两条客户端路径，src/services/compact/apiMicrocompact.ts 还提供一条让 API 自己清理的路径——不在客户端动消息，而是在请求体里塞 context_management 的 edits 策略数组，让服务端在打 prefix cache 时顺便做清理。

两条独立策略：第一条是 clear_thinking_20251015，默认启用范围是所有带 thinking 的请求，每次请求都声明。行为上，keep 为 all 时默认保留所有 thinking；keep 指定 thinking_turns 为 1 时，仅保留最近一轮。第二条是 clear_tool_uses_20250919，默认仅对 Anthropic 内部员工启用，即 USER_TYPE 为 ant，并需要 USE_API_CLEAR_TOOL_RESULTS 或 USE_API_CLEAR_TOOL_USES 环境变量开启。触发条件是 input_tokens 超过 180K，可用 API_MAX_INPUT_TOKENS 覆盖；行为是清理旧的 tool_result 内容或旧的 tool_use 本体，clear_at_least 保证至少清出 140K。

clearAllThinking 为 true 时何时锁存？位置在 src/services/api/claude.ts 第 1446 到 1455 行：

（代码从略：先读取已有的锁存状态。如果尚未锁存且当前是 agentic 查询，就取上一次 API 完成的时间戳；如果当前时间距它超过一小时的缓存 TTL，就把 thinking 清理锁存置为真并保存。）

注释直白地道出意图——距上次 API 完成超过 1 小时，缓存 TTL 必过期，prefix 反正要重写，趁机多清点 thinking blocks。一旦锁存，整个会话都保持该状态，与 3.6 节第二层的粘性锁存一脉相承，避免缓存键中途翻转。

设计决策：为什么 clear_tool_uses_20250919 要设置 clear_at_least 为 140K？

官方文档的 Context editing 说明明确指出：清理工具结果会使被清内容所在的缓存前缀失效，为此要清理足够多的 token，让缓存失效物有所值。清理工具结果会直接破坏 prompt cache 前缀——这意味着本次请求的缓存写入成本，即 cache_creation_input_tokens，必然发生。如果只清掉几千 token，付出了一整次缓存重写的代价却没换回多少上下文空间，得不偿失。clear_at_least 设为 140K 确保每次触发都清出足够量，即 180K 触发阈值减 40K 目标等于 140K，让缓存失效的代价换来实质性的空间收益。

与此形成对比的是 clear_thinking_20251015：官方文档说明，当 thinking blocks 保留在上下文中时，prompt cache 得以保全——保留 thinking blocks 不破坏缓存，所以不需要 clear_at_least 约束。但当 thinking blocks 被清除，即 clearAllThinking 为 true、只保留最近一轮 thinking 时，同样会破坏缓存；只是那时已经是空闲 1 小时、等到缓存 TTL 过期后才触发，缓存反正要重写，所以也不需要特别保底。

Level 4：Context Collapse

投影式上下文折叠——关键特性是它不修改原始消息。它创建消息的折叠视图，将不重要的早期消息替换为摘要。这使得折叠可以跨轮次持久化，且可以在需要时回退。

（代码从略：src/query.ts 中，如果 CONTEXT_COLLAPSE 功能开启且 contextCollapse 可用，就调用 applyCollapsesIfNeeded 按需应用折叠，并用返回的消息替换 messagesForQuery。要点是 Context Collapse 是读时投影，不写原始消息。）

可以用数据库的 View 来类比：底层表，即消息数组，数据不变，但查询时，也就是发送 API 请求时，看到的是一个过滤或转换后的视图。摘要存储在独立的 collapse store 中，projectView 在每次循环入口将折叠视图叠加到原始消息之上。

以 effectiveWindow 为同一分母来看：Context Collapse 在约 90% 利用率时提交折叠、约 95% 时阻塞并派生子任务，而 Autocompact 在约 93% 触发，即 effective 减 13k。源码注释明确指出了这个竞争关系：autocompact 的触发点正好卡在 collapse 的提交起点 90% 和阻塞点 95% 之间，会与 collapse 竞争并通常获胜，破坏 collapse 本要保存的细粒度上下文。因此，当 Context Collapse 启用且活跃时，Autocompact 被抑制。

Level 5：Autocompact

这是最后的手段——当所有轻量级压缩都无法将 Token 使用量控制在安全范围内时，系统 fork 一个子 Agent 来生成整个对话的摘要。

但是"最后手段"也有两条子路径——按代价由低到高：

5a，Session Memory Compact，省去压缩时刻的 LLM 调用。

src/services/compact/sessionMemoryCompact.ts 的 trySessionMemoryCompaction，会在 compactConversation 之前，先尝试用 Session Memory 文件替代"老消息的摘要"。这需要 Anthropic 通过 GrowthBook 同时开启两个 flag：tengu_session_memory 控制是否生成 Session Memory 文件；tengu_sm_compact 控制是否将该文件用于压缩替代。GrowthBook 是 Anthropic 内部使用的 feature flag 服务，类似 LaunchDarkly。这是 Anthropic 服务端的灰度策略，用户无法自行开启，但源码中用 ENABLE_CLAUDE_CODE_SM_COMPACT 和 DISABLE_CLAUDE_CODE_SM_COMPACT 两个环境变量提供了手动覆盖，仅测试用。

注意命名歧义：Session Memory 不等于持久记忆系统，也就是 memdir 下的 MEMORY.md。Session Memory 的路径是项目目录下、会话 ID 目录中的 session-memory 目录里的 summary.md——带会话 ID，只在本会话生效；memdir 才是用户主目录 .claude/projects 下按项目 slug 存放的 memory/MEMORY.md 那种跨会话持久化。两者都叫 memory 但完全独立：memdir 是跨会话的长期事实存储，session memory 是当前会话持续维护的结构化工作笔记。

Session Memory 由后台 fork agent 周期性抽取，写入一个固定 10 个 section 的 markdown 模板，包括 Current State、Task specification、Files and Functions、Errors and Corrections、Learnings、Worklog 等，见 src/services/SessionMemory/prompts.ts。触发抽取的条件位于 sessionMemoryUtils.ts 第 32 到 38 行：上下文达到 10K token 才初始化，之后每增长至少 5K token 且至少 3 次工具调用才更新一次——保证后台 fork agent 不喧宾夺主。

5a 子路径的输出：保留尾部 10K 到 40K token 的原文，把前面的段落用 summary.md 的内容包成 isCompactSummary 消息替代。sessionMemoryCompactConfig 的默认值：

（代码从略：三个默认值。minTokens 为 10000，即至少保留 10K token 的最近消息原文；minTextBlockMessages 为 5，即至少 5 条含文本块的消息；maxTokens 为 40000，即最多保留 40K。）

adjustIndexToPreserveAPIInvariants 还会修正 startIndex，有两点：第一，不切散 tool_use 和 tool_result 的配对，孤儿的 tool_result 会导致 422 错误；第二，不切散流式阶段同一 message.id 但不同 uuid 的 thinking 加 tool_use，合并后会丢失 thinking。

回退守卫：如果 summary.md 还是初始模板、从未填充过，sessionMemoryCompact.ts 第 442 到 446 行会直接回退到下面的 5b 全量 LLM 摘要路径。

5b，全量 LLM 摘要，即 compactConversation：经典路径，下面的章节详解。

触发条件由 shouldAutoCompact 判断——必须同时满足 5 个条件：

（代码从略：第一，递归守卫，查询来源不能是 session_memory 或 compact，防止压缩 Agent 自己触发压缩，造成死循环。第二，三重开关检查，isAutoCompactEnabled 会检查 DISABLE_COMPACT 环境变量、DISABLE_AUTO_COMPACT 环境变量和 userConfig 中的 autoCompactEnabled 设置，任一禁用则不触发。第三，非 Reactive-only 模式，当 REACTIVE_COMPACT 启用且特定标志活跃时，让 API 自己的 PTL 错误触发反应式压缩，而非主动压缩。第四，非 Context Collapse 模式，当 CONTEXT_COLLAPSE 启用且活跃时，autocompact 被抑制；原因是 collapse 在约 90% 提交，autocompact 在约 93% 触发，两者阈值接近会竞争，autocompact 可能销毁 collapse 正要保存的细粒度上下文。第五，Token 阈值，含 snipTokensFreed 修正，即估算的 token 数减去 snip 释放量，要达到 getAutoCompactThreshold 给出的模型阈值。）

阈值计算：

（代码从略：源码中 getEffectiveContextWindowSize 的算法是，摘要预留 token 数 reservedTokensForSummary 取模型最大输出 token 与 MAX_OUTPUT_TOKENS_FOR_SUMMARY 的较小值。后者为 20000，基于 p99.99 摘要输出 17387 tokens 得出；模型最大输出 token 受 slot-cap feature flag 影响，开启时为 8K。有效窗口等于上下文窗口减去摘要预留。然后 getAutoCompactThreshold 的算法是，autocompact 阈值等于有效窗口再减 AUTOCOMPACT_BUFFER_TOKENS，即 13000 的缓冲。）

以 200K 上下文窗口为例，实际阈值取决于 reservedTokensForSummary，分两种场景。场景一，slot-cap 开启，模型最大输出为 8K，摘要预留取 8K 和 20K 的较小值，即 8K；有效窗口是 192000，autocompact 阈值是 179000，约为总窗口的 89.5%。场景二，slot-cap 关闭，模型最大输出不小于 20K，摘要预留就是 20K；有效窗口是 180000，阈值是 167000，约为总窗口的 83.5%。

可通过 CLAUDE_AUTOCOMPACT_PCT_OVERRIDE 环境变量按百分比覆盖。

压缩提示词的设计，位于 src/services/compact/prompt.ts：

压缩的质量取决于给子 Agent 的提示词。Claude Code 在这里使用了一个精巧的"分析-摘要"两阶段模式：

首先，一个激进的 NO_TOOLS_PREAMBLE 确保摘要模型不会尝试调用工具。在 Sonnet 4.6 以上的自适应思考模型上，模型有时会无视较弱的限制而尝试工具调用，导致无文本输出。

（代码从略：这段前导的要点是，用强烈的措辞要求模型只用文本回复、绝不调用工具，因为工具调用会被拒绝，并且会浪费唯一一轮机会，导致任务失败。）

然后，模型被要求生成两个部分：

第一部分是 analysis 块，即思考草稿，按时间顺序分析对话中的每条消息：用户的意图、采取的方法、关键决策、文件名、代码片段、错误及修复、用户反馈。

第二部分是 summary 块，即正式摘要，包含 9 个标准化部分。第 1 部分，Primary Request，用户的所有显式请求和意图。第 2 部分，Key Technical Concepts，讨论的技术概念和框架。第 3 部分，Files and Code，检查、修改或创建的文件及关键代码片段。第 4 部分，Errors and Fixes，遇到的错误及修复方式，特别是用户反馈。第 5 部分，Problem Solving，已解决的问题和进行中的排查。第 6 部分，All User Messages，所有非工具结果的用户消息原文。第 7 部分，Pending Tasks，待完成的任务。第 8 部分，Current Work，压缩前正在进行的工作，要求最详细。第 9 部分，Optional Next Step，下一步计划，包含原始对话的直接引用。

关键的设计巧思：formatCompactSummary 会剥离 analysis 块，只保留 summary 进入上下文。这是经典的"链式思考草稿"技术，英文叫 Chain-of-Thought Scratchpad——让模型先推理再总结，质量远超直接生成摘要，但推理过程本身如果保留在上下文中会浪费大量 Token。丢弃分析、保留结论，两全其美。

压缩后恢复机制：

Autocompact 的风险是让模型"忘记"刚编辑的文件。系统会在压缩后自动执行 runPostCompactCleanup：

流程图描述如下。首先，Token 达到阈值后触发，先做递归守卫检查，确认当前不在 compact 查询源中。然后先尝试 5a 的 Session Memory Compact，即 trySessionMemoryCompaction，用 summary.md 替代老消息摘要，省去压缩时刻的 LLM 调用；再进入 5b 的全量 LLM 摘要，由 compactConversation fork 子 Agent 生成摘要。最后执行压缩后清理 runPostCompactCleanup，它做四件事：恢复最近 5 个文件，每个不超过 5K Token；恢复已调用的技能，不超过 25K Token；重置 context-collapse；重新通告延迟工具，包括 Agent 列表和 MCP 指令。

具体展开是五项：

第一，恢复最近 5 个文件：从压缩前的 readFileState 缓存中取出最近读取的 5 个文件，每个限 5K Token，作为附件消息注入。

第二，恢复所有已激活的技能：预算 25K Token，每个技能限 5K Token，确保已加载的技能不丢失。

第三，重新通告上下文增量：压缩吃掉了之前的延迟工具、Agent 列表、MCP 指令等增量通告，重新从当前状态生成。

第四，重置 Context Collapse：清除折叠状态，为下一轮压缩准备。

第五，恢复 Plan 状态：如果当前在 Plan 模式或有活跃计划，注入相关指令。

这个恢复机制是 Claude Code 能在超长对话中保持连贯性的关键。没有它，模型在压缩后会忘记自己刚才编辑了哪些文件，导致后续操作可能重复读取或产生不一致的修改。

压缩请求本身可能超限：当对话已经极长时，发送完整消息让子 Agent 摘要的请求本身也可能触发 Prompt-Too-Long 错误。truncateHeadForPTLRetry 通过按 API 轮次分组、从头部丢弃最旧的轮次来缩小压缩请求，最多重试 3 次。

熔断器机制：连续 3 次 autocompact 失败，即达到 MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES，就停止重试——上下文不可恢复地超限。这个熔断器来自真实数据：曾有 1279 个会话连续失败超过 50 次，最高 3272 次，浪费了每天约 250K 次 API 调用。

关键常量如下：

AUTOCOMPACT_BUFFER_TOKENS，值为 13000，用作触发阈值缓冲。WARNING_THRESHOLD_BUFFER_TOKENS，值为 20000，用作 UI 警告阈值。ERROR_THRESHOLD_BUFFER_TOKENS，值为 20000，用作阻塞限制阈值。MANUAL_COMPACT_BUFFER_TOKENS，值为 3000，用作手动压缩缓冲。MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES，值为 3，是熔断器阈值。DEFAULT_MAX_RESULT_SIZE_CHARS，值为 50000，是工具结果持久化阈值。PREVIEW_SIZE_BYTES，值为 2000，是持久化结果的预览大小。POST_COMPACT_MAX_FILES_TO_RESTORE，值为 5，是压缩后恢复文件数。POST_COMPACT_MAX_TOKENS_PER_FILE，值为 5000，是每个恢复文件的 Token 上限。POST_COMPACT_SKILLS_TOKEN_BUDGET，值为 25000，是技能恢复总预算。

3.5 Token 预算管理

Claude Code 维护精细的 Token 预算追踪：

输出 Token 预留，按模型区分

getModelMaxOutputTokens 位于 src/utils/context.ts，按模型返回默认值和上限：

Opus 4.6 的默认 max_output_tokens 是 64000，上限是 128000。Sonnet 4.6 默认 32000，上限 128000。Opus 4.5、Sonnet 4.x 和 Haiku 4 默认 32000，上限 64000。Opus 4 和 4.1 默认 32000，上限 32000。Claude 3.7 Sonnet 默认 32000，上限 64000。Claude 3 Sonnet 以及 3.5 Sonnet 和 Haiku 默认 8192，上限也是 8192。Claude 3 Opus 和 Haiku 默认 4096，上限也是 4096。

未命中的模型走兜底 MAX_OUTPUT_TOKENS_DEFAULT；可通过 CLAUDE_CODE_MAX_OUTPUT_TOKENS 环境变量覆盖。思考 Token 预算不再是一张固定表：getMaxThinkingTokensForModel 已标记 deprecated，实现只是返回 upperLimit 减 1，新模型改用自适应思考，即 adaptive thinking，而非严格的思考预算。

Token 估算算法

tokenCountWithEstimation 位于 src/utils/tokens.ts，是上下文大小估算的核心函数。它的设计原则是从不调用 API——避免网络延迟对压缩决策的影响。

核心思路可以用一个类比来理解：假设你今早称了体重是 75 公斤，此后吃了一顿午饭。你不需要再次上称——估计 75.5 公斤就足够好了。tokenCountWithEstimation 的"体重秤"是 API 返回的 usage 数据，即服务端精确计算的 Token 数；"午饭"是此后新增的少量消息。

算法逻辑：

（代码从略：tokenCountWithEstimation 函数分四步。第一步，从消息末尾向前查找，找到最近一条带有 API usage 数据的消息。第二步，向前跳过同一 API 响应的分裂记录，也就是相同 message.id 的消息，因为并行工具调用可能把一个响应拆成多条消息；过程是锚定到同一响应的最早记录，遇到不同的 API 响应就停止。第三步，用服务端报告的 token 数作为锚点，加上锚点之后消息的粗略估算，作为返回值。第四步，如果没有任何 usage 数据，例如会话刚开始，就完全靠字符串长度估算。）

关键洞察：每次 API 响应都自带 usage 数据，包含 input_tokens、output_tokens 和 cache tokens，这是服务端精确计算的结果。tokenCountWithEstimation 把这个精确值作为锚点，只对锚点之后的新消息，通常只有几条工具结果，做粗略估算。估算按字符数除以 4，即每 token 约 4 字节；密集 JSON 里单字符 token 多，改用除以 2 的更保守比率。

这比完全靠客户端估算精确得多，误差从可能的 30% 以上降到通常小于 5%，同时又不需要额外的 API 调用。

Task Budget 跨压缩结转

每次压缩前捕获 finalContextTokensFromLastResponse，压缩后从剩余量中扣除。这确保跨压缩的 Token 预算连续性——压缩会"替换"消息，但服务端看到的只是压缩后的摘要，不知道压缩前的上下文有多大。taskBudgetRemaining 告诉服务端：这些 Token 已经被"花掉"了。

（代码从略：query.ts 中维护一个循环级别的 taskBudgetRemaining 变量，初始为未定义；每次 compact 时，把它减去 finalContextTokensFromLastResponse 对 messages 的计算结果。）

3.6 前缀缓存策略

前面提到，上下文工程是"带着镣铐跳舞"——镣铐就是前缀缓存。这条镣铐是什么形状，Claude Code 又如何在它的约束下把舞跳好，下面细看。

为什么需要前缀缓存？

每次 API 请求，服务端都需要对输入做一遍 KV Cache 计算，这是 transformer 注意力机制的底层操作。一次请求的完整输入可能有 100K 到 200K token——如果每次都从头算，延迟和成本都不可接受。

前缀缓存的原理：服务端记住上一次请求的 KV Cache 结果，下一次请求时，如果前缀完全一致，就直接复用之前的计算结果，只需要处理新增的部分。但这里有一个 transformer 架构层面的硬约束：前缀必须字节级完全一致才能复用 KV Cache。不是"差不多就行"，而是任何一个字节的变化都会导致该位置之后的所有 KV Cache 失效。

缓存链：从系统提示词到对话内容

Claude Code 并不只是缓存系统提示词——它在 system 和 messages 两个层次显式设置缓存断点，串成一条完整的缓存链；工具数组因为渲染在 system 之前，被 system 的断点一并覆盖。

一张示意图对比了两个连续请求。请求 N 中，system 块标记了 cache_control，工具数组随 system 一并缓存，system 与 tools 这两段都在缓存命中范围内；历史消息 msg1、msg2、msg3 的末尾也标记了 cache_control。到了请求 N+1，system 和 tools 依旧命中，msg1 到 msg3 的历史也整体命中，只有新增的 msg4、msg5 需要计算，cache_control 标记随之移到最新的消息上。

第一个断点：系统提示词。splitSysPromptPrefix 在系统提示词的静态与动态边界处标记 cache_control，使得核心指令部分可以跨用户共享缓存，详见下文的"分割策略"。API 的渲染顺序是先 tools、再 system、最后 messages，标在 system 块上的断点会把它前面的整个工具数组一并纳入缓存前缀——所以工具定义不必单独打断点也能被缓存。

第二个断点：消息数组。addCacheBreakpoints 在最后一条消息上标记 cache_control。这意味着所有历史消息，即上一轮及之前的，都在缓存前缀内，每轮只需处理新增的消息。

关于工具数组的断点，还有一个细节：toolToAPISchema 保留了给工具 schema 打 cache_control 的能力，即 options.cacheControl 参数，但在我们分析的这份快照里，主查询路径没有任何调用传入它，claude.ts 那里只传了 deferLoading，源码里"工具 schema 携带缓存标记"那句注释描述的是意图、并未落地。所以实际生效的是 system 与 messages 两个断点，工具由前者覆盖。可选的服务端工具，例如 advisor，排在工具数组末尾——按设计意图，工具断点应打在它们之前、让开关不破坏缓存；工具断点既然没接线，这层保护在快照里也就没有生效。

两个断点串联起来，实现了全链路缓存：一个持续 20 轮的对话，第 21 轮只需要处理最新的用户消息和工具结果，前面积累的 system、tools 加 20 轮历史消息全部命中缓存。这是 Claude Code 能保持低延迟响应的核心原因之一。

细节：fire-and-forget 请求的缓存保护。某些次要请求，例如后台的辅助查询，会把 cache_control 标记在倒数第二条消息上，而不是最后一条。这样临时请求不会把自己的内容写入主缓存链，避免污染后续正常对话的缓存前缀。

细节：Assistant 消息的特殊处理。cache_control 只标记在消息的最后一个内容块上，但会跳过 thinking 和 redacted_thinking 块——这些块的内容不稳定，标记在它们上面会降低缓存命中率。

缓存稳定性：四层防御

全链路缓存的收益巨大，但也意味着缓存失效的代价极高——一次意外断裂，就可能让 100K 以上的 token 前缀全部重新计算。Claude Code 建立了四层防御来维护缓存稳定性。

第一层：系统提示词分割——最大化缓存共享。

核心问题：系统提示词既包含全用户通用的内容，如核心指令、安全规则、工具描述；也包含因用户而异的内容，如 CLAUDE.md 引用、MCP 工具指令、输出风格偏好。如果整个系统提示词只能作为一个整体缓存，那么每个用户的缓存都不同——数百万用户就需要数百万份缓存，其中大部分内容是完全重复的。

解决方案：splitSysPromptPrefix 位于 src/utils/api.ts，它通过 SYSTEM_PROMPT_DYNAMIC_BOUNDARY 标记，将系统提示词在通用和专属的边界处切开，对两部分使用不同的缓存策略。

但这里有一个决策树，因为不是所有场景都能使用最优策略：

第一问，用户安装了 MCP 工具吗？如果是，MCP 工具的 schema 因用户而异，因为不同用户安装了不同的 MCP server。即使系统提示词文本相同，工具数组也不同——全局缓存无法生效。此时所有块降级为 org 档，即组织内共享。

第二问，全局缓存功能可用且边界标记存在吗？如果是，这是最优路径：边界之前的静态内容使用 global 档，全球所有用户共享同一份缓存；边界之后的动态内容不缓存。

第三，以上都不满足？回退为所有内容使用 org 档。

三种场景汇总如下。场景一，有 MCP 工具：静态内容和动态内容都用 org 缓存，共享范围是组织内。场景二，无 MCP 且支持全局缓存：静态内容用 global 缓存，动态内容不缓存，共享范围是全球。场景三，回退情况：静态和动态都用 org 缓存，共享范围是组织内。

第二层：会话级锁存——防止中途翻转。

即使分割做对了，缓存仍然可能因为元数据的中途变化而失效。有两类特别隐蔽的风险：

风险一：缓存过期时间变化。

服务端缓存有过期时间，即 TTL：普通用户是 5 分钟，内部用户和付费订阅用户可以享受 1 小时。但 TTL 本身也是缓存键的一部分——如果一个用户在对话到第 10 轮时用量超标，系统将其从 1 小时降级为 5 分钟，缓存键就变了，之前积累的缓存全部作废。

Claude Code 的解法：在会话开始时锁定缓存资格，中途不变。setPromptCache1hEligible 在首次 API 调用时评估用户是否有资格使用 1 小时 TTL，结果缓存到会话结束。即使用户中途超标，TTL 也不会降级。

风险二：API 请求头变化。

Claude API 支持一些 beta 功能，例如 fast-mode、cache-editing、thinking-clear 等，通过 HTTP 请求头启用。这些请求头也会影响服务端的缓存键——如果第 3 轮请求带了 fast-mode header，第 4 轮没带，缓存键就不同了。

而这些功能可能受 feature flag 控制，flag 随时可能在服务端切换。想象一下：用户正在编码，feature flag 突然关闭了 fast-mode，下一次请求就少了一个 header，50 到 70K token 的缓存立刻作废。

Claude Code 的解法：一旦某个 beta header 被首次发送，它就会在该会话的所有后续请求中持续发送——即使触发它的 feature flag 已经关闭。源码中管这叫"粘性锁存"，英文是 sticky-on latch，涉及四个 header：AFK 模式、快速模式、缓存编辑、思考清理。

两种锁存共享一个重置点：clear 和 compact 命令会同时重置所有锁存状态。这是合理的——这两个命令本身就会重建对话上下文，缓存必然要重新构建，锁存也就没有保护的必要了。

设计权衡：锁存牺牲了灵活性，无法中途关闭某个 beta 功能、无法中途降级 TTL，换来了缓存稳定性。在 100K 以上 token 缓存失效的高成本面前，这是值得的取舍。

第三层：工具数组与消息数组的排列策略。

工具排列：设计意图是把可选的服务端工具，例如 advisor，排在工具断点之后，让 advisor 的开关只改变断点之后的一小段内容。不过如 3.6 节所述，快照中工具断点并未接线，这层保护没有实际生效。MCP 工具通过 defer_loading 延迟加载机制，在 Tool Search 被调用之前不出现在工具数组中——与第一层的分割策略配合，减少因用户特有工具导致的缓存差异。

Cached Microcompact 的缓存感知：前面 3.4 节提到 Microcompact 有两条路径。当缓存是热的、未过期时，它不直接修改消息内容，因为那会破坏整个消息前缀的缓存，而是通过 cache_edits 指令告诉服务端在缓存中把某些 tool_result 块删掉。这样既清理了旧内容释放了空间，又不需要客户端重新上传和服务端重新处理整个前缀——这是消息层缓存和压缩机制之间的精妙配合。

第四层：缓存断裂检测——诊断安全网。

有了前三层防御，缓存应该是稳定的。但"应该"和"实际"之间总有差距——Claude Code 建立了一个诊断系统来验证这些防御是否真正生效。

promptCacheBreakDetection 实现了两阶段快照对比：

第一阶段，API 调用前：记录当前所有影响缓存键的状态——系统提示词 hash、工具 schema hash，精确到每个工具，还有 beta header 列表、TTL 设置等。

第二阶段，API 调用后：检查响应中的 cache_read_input_tokens。如果比上次下降超过 5% 且 2000 token，判定为缓存断裂。

检测到断裂后，系统自动归因到三种原因之一：

原因一，TTL 过期：距上次请求超过了缓存时间窗口，即用户离开太久。

原因二，客户端变更：通过对比前后的 hash 差异，精确定位是哪个字段变了，甚至能定位到具体是哪个工具的 schema 变了。

原因三，服务端驱逐：客户端一切不变，但缓存仍然失效——说明是服务端主动驱逐了缓存。

这个检测系统形成了一个改进闭环：源码注释里提到，第二层的锁存机制正是在检测系统发现特定 header 翻转打断缓存后才补上的。先有检测，发现问题，再建防御——这是工程上的正循环。

小结：Claude Code 的前缀缓存是一个全链路方案——system 与 messages 两层显式打了缓存断点，工具数组随 system 断点一并缓存，串联成一条完整的缓存链。四层防御机制，即分割、锁存、排列策略、断裂检测，共同确保这条缓存链的稳定性。最终效果：无论对话进行到第几轮，每次请求只需处理最新的增量内容，前面积累的所有上下文都由 KV Cache 免费提供。

3.7 system-reminder 注入机制

Claude Code 需要在对话的各个位置注入系统级信息——当前可用的延迟工具列表、记忆文件内容、安全提醒等。但直接插入这些内容会产生一个问题：模型可能误认为这是用户说的话，从而做出不恰当的响应。

system-reminder 是解决这个问题的统一机制。系统提示词中有明确说明：工具结果和用户消息中可能包含 system-reminder 标签。它们包含由系统添加的有用信息和提醒，与它们所出现的具体工具结果或用户消息无关。

注入位置有三种：

第一种，用户上下文前置，由 src/utils/api.ts 中的 prependUserContext 实现。

CLAUDE.md 内容、当前日期等被包装在 system-reminder 标签中，作为第一条 isMeta 用户消息插入：

（代码从略：创建一条 isMeta 为 true 的用户消息，内容是 system-reminder 包裹的上下文，大意是：回答用户问题时可以使用以下上下文；然后是 claudeMd 部分，放 CLAUDE.md 的内容；接着是 currentDate 部分，写明今天的日期；最后附一句说明，这些上下文可能与你的任务相关，也可能无关。）

第二种，附件消息：记忆预取结果，详见 3.8 节记忆预取；延迟工具列表，即 Tool Search 的发现结果；还有技能列表、Agent 定义列表等，都作为附件消息注入，内容包裹在 system-reminder 中。

第三种，工具结果中的提醒：某些工具在返回结果时附带系统提醒。例如：文件读取发现文件为空时，警告文件存在但为空；文件读取偏移超过文件长度时的提醒；MCP 资源访问后的安全边界提醒。

为什么用 XML 标签？

XML 标签创建了一个清晰的语义边界。模型通过训练知道 system-reminder 内的内容是系统自动注入的元数据，而不是用户的直接输入。这使得系统可以在对话的任意位置注入上下文——工具结果之后、用户消息之间——而不会混淆消息的"发言者"身份。

同时，消息规范化中的 smooshSystemReminderSiblings 步骤会将相邻的 system-reminder 文本块合并到邻近的 tool_result 中，避免产生多余的 Human 和 Assistant 轮次边界。

3.8 记忆预取

记忆预取是 Claude Code 在模型生成响应的同时，并行搜索相关记忆文件的优化机制。它的核心价值是隐藏延迟——搜索记忆文件需要磁盘 I/O，与其等模型响应完再搜索，即串行，不如在模型思考的同时就开始搜索，即并行。

startRelevantMemoryPrefetch 位于 src/utils/attachments.ts，在每次 query 循环迭代入口启动：

（代码从略：src/query.ts 中，用 using 关键字启动记忆预取并确保 dispose：把 state.messages 和 state.toolUseContext 传给 startRelevantMemoryPrefetch，结果赋给 pendingMemoryPrefetch。）

工作流程有六步：

第一步，启动条件：isAutoMemoryEnabled 为 true 且相关 feature flag 活跃。

第二步，并行执行：在 callModel 流式调用期间并行运行，搜索本项目的自动记忆目录，即用户主目录 .claude/projects 下、按项目路径转义命名的 memory 目录，同一 git 仓库的所有 worktree 共享一份，可被 settings 或环境变量覆盖，从中寻找与当前对话相关的记忆文件。

第三步，单次消费：通过 settledAt 守卫确保每轮只消费一次。如果 query 循环因 PTL 恢复而重试，预取结果不会被重复注入。

第四步，去重：readFileState 追踪已读文件，防止同一个记忆文件在同一会话中被多次注入。

第五步，注入时机：预取结果作为附件消息，即 AttachmentMessage，在工具执行之后注入，出现在下一轮 API 调用的上下文中。

第六步，资源清理：using 语法确保在 generator 退出时，无论正常、异常还是中断，都自动调用 dispose，发送遥测数据并清理资源。

3.9 反应式压缩

当 Prompt-Too-Long 错误发生时，简称 PTL，反应式压缩作为最后手段触发：

（代码从略：src/query.ts 中 PTL 恢复的第二阶段。reactiveCompact 由 REACTIVE_COMPACT 这个 feature 开关门控引入，实现位于 services/compact/reactiveCompact.ts，但本源码快照中缺失。当错误是 413 或媒体被扣留，且 reactiveCompact 可用时，调用 tryReactiveCompact，传入是否已尝试过反应式压缩、查询来源、中止信号、查询消息，以及缓存安全参数，包括系统提示词、用户上下文、系统上下文、工具上下文和 fork 上下文消息。如果压缩成功，就构建压缩后消息，把状态转移原因设为 reactive_compact_retry，继续循环。）

在正常运行中，autocompact 应该在上下文利用率达到阈值时主动触发，约占总窗口的 83% 到 90%，取决于模型的 reservedTokensForSummary，从而防止 PTL 错误发生。反应式压缩只在以下情况下需要：autocompact 被禁用或跳过；单次工具结果异常大，一步跳过了 autocompact 阈值；Context Collapse 排水释放的 Token 不够。

设计决策：压缩阈值是怎么确定的？

自动压缩的触发公式是：token 数大于等于有效上下文窗口减去 autocompact 缓冲，缓冲常量 AUTOCOMPACT_BUFFER_TOKENS 为 13000，位于 src/services/compact/autoCompact.ts。有效上下文窗口本身等于上下文窗口减去模型最大输出 token 与 20000 的较小值，也就是扣除了压缩摘要的输出预留；MAX_OUTPUT_TOKENS_FOR_SUMMARY 为 20000，依据是 p99.99 的压缩摘要输出为 17387 tokens。对于 200K 上下文窗口，阈值相对有效窗口约 92.8%，即 167K 比 180K；相对总窗口约 83.5% 到 89.5%，取决于摘要预留是 20K 还是 8K。13K 缓冲确保触发压缩时还有足够空间完成当前工具执行和生成摘要。与此相关的还有 WARNING_THRESHOLD_BUFFER_TOKENS 为 20000：警告阈值等于 autocompact 阈值减 20000，即在自动压缩触发前 20K token 就开始向用户显示警告；autocompact 关闭时，则相对有效窗口减 20K。

设计决策：为什么 max_output_tokens 默认只用 8K 而不是 32K？

CAPPED_DEFAULT_MAX_TOKENS 为 8000，位于 src/utils/context.ts。源码注释解释了原因：BQ 的 p99 输出是 4911 tokens，所以 32k 或 64k 的默认值会过度预留 8 到 16 倍的 slot 容量。API 服务端会根据 max_output_tokens 预留计算资源，即 slot；如果每个请求都声明 32K 但实际只用 5K，服务端的资源利用率极低。8K 作为默认值覆盖了 99% 的实际需求。当模型确实因为 max_tokens 截断时，系统自动升级到 ESCALATED_MAX_TOKENS，即 64000，并清洁重试——这就是 MOT 恢复机制，MOT 指的是 Max Output Tokens。

3.10 实践指南：如何高效利用 KV Cache

理解了前缀缓存架构，反过来看：作为用户，哪些习惯能最大化缓存命中率，让响应更快、成本更低；哪些操作会无意中"打碎"缓存？

缓存友好的使用习惯

第一，保持对话连续性，避免长时间中断。

普通用户的缓存 TTL 是 5 分钟，付费订阅用户是 1 小时。这意味着如果你离开超过 5 分钟再回来发消息，服务端的 KV Cache 已经过期——整个前缀，即 system、tools 加所有历史消息，需要从头计算。你会明显感觉到第一次回复变慢。

实践建议：如果需要短暂离开思考，尽量控制在 5 分钟内回来继续对话。如果预计要离开较久，接受回来后第一轮会稍慢——这是 TTL 过期的正常现象，后续轮次会立即恢复正常速度。

第二，如果是具有相关性的任务下，长对话优于频繁新建会话。

每次新建会话都是一次完全的冷启动——50 到 100K token 的系统提示词和工具定义需要从头处理。而在同一个会话中继续对话，这些前缀都已经在缓存中了，每轮只需处理新增的消息。

实践建议：尽量在同一个会话中完成相关工作，而不是为每个小任务都新开一个会话。如果你同时有多个 Claude Code 会话，会话之间的缓存也是独立的——它们不能共享消息历史的 KV Cache，系统提示词部分的缓存可以跨会话共享。

第三，让自动压缩替你管理上下文。

Claude Code 内置了五级压缩流水线，会在上下文接近窗口限制时自动触发。你不需要手动干预——系统知道最佳的压缩时机和策略。

实践建议：看到上下文使用率警告时不必紧张，让系统自动处理即可。只在你明确想"重新开始一段相关的对话逻辑"时才手动使用 compact 命令；如果相关的逻辑可以直接用 clear 命令。

第四，精简 MCP 工具安装。

这是很多用户不知道的：只要有 MCP 工具实际渲染进工具数组，即未被 defer_loading 延迟加载，整个系统提示词的缓存就会从 global 档，也就是全球共享，降级为 org 档，也就是组织内共享。这意味着你无法享受全球数百万用户共享的系统提示词缓存——每次冷启动都需要独立计算。被 Tool Search 延迟加载的 MCP 工具不会触发降级，见 3.6 节。

实践建议：只安装你真正在用的 MCP server。如果某个 MCP server 只是偶尔用一次，考虑用完后移除。MCP 工具越少，缓存效率越高。

还有一点：频繁切换模型。

不同模型在服务端使用不同的 KV Cache 空间。如果你在同一个会话中频繁切换模型，比如从 Opus 切到 Sonnet 再切回来，每次切换都无法命中之前模型的缓存。

实践建议：在一个会话中尽量使用同一个模型。如果需要切换，可以考虑开一个新会话。

缓存效率的心智模型

最后，用一个简单的心智模型来总结：你的每次请求，等于已缓存的前缀，加上新增的内容。前缀部分是免费的，因为 KV Cache 已经存在；新增部分需要计算，消耗时间和成本。

你的目标是最大化"已缓存的前缀"部分。做到这一点的核心原则就是保持稳定性——保持会话连续、避免不必要的重置、减少会改变前缀的操作。前缀越稳定，缓存命中率越高，响应越快，成本越低。

3.11 设计洞察

第一，Memoize 保证幂等性：getSystemContext 和 getUserContext 都是 memoized 的，每会话只计算一次。setSystemPromptInjection 变更时会手动清除两个函数的缓存。

第二，压缩流水线的渐进性：从零成本裁剪到全量摘要，按需逐级升级。大部分对话永远不会触发 Autocompact。

第三，投影式折叠的可逆性：Context Collapse 不修改原始消息，可以安全回退——这是它优于 Autocompact 的地方。

第四，缓存感知的上下文组装：上下文的注入顺序，即系统提示词在前、用户上下文在消息前，考虑了提示词缓存的命中率。

第五，Token 估算的锚点策略：用服务端报告的精确 usage 作为锚点，只估算增量，在精度和延迟之间取得平衡。

第六，system-reminder 作为统一注入通道：通过 XML 标签包装，在消息流的任意位置注入系统信息，而不混淆角色边界。

第七，会话级锁存的务实取舍：TTL 资格和 beta header 一旦确定就锁定到会话结束，牺牲灵活性换取缓存稳定性——50 到 100K token 缓存失效的代价，远高于中途无法切换某个功能。

动手实践：在 claude-code-from-scratch 项目中，src/prompt.ts 和 src/system-prompt.md 展示了最小实现的上下文构建方式。对比本章的多层上下文组装，思考：一个最小 Agent 需要哪些上下文就够用了？参见教程的第 3 章：System Prompt 工程。

上一章是系统主循环，下一章是工具系统。
