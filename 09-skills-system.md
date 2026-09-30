---
title: 第 5 章：技能系统（朗读版）
---

第 5 章：技能系统

本文是 docs/09-skills-system.md 的朗读版，表格与代码已转为口语描述，内容未增删。

技能是 Claude Code 的"AI Shell 脚本"——将验证有效的 prompt 模板化，让 Agent 不必每次从头编写相同的流程。

5.1 什么是技能？

Shell 脚本自动化终端任务，技能自动化 AI 任务。拆开看，一个技能就是三样东西：提示词模板、元数据、执行上下文。

首先，一个技能整体上就是一个 Markdown 文件。然后它分成两部分。第一部分是 Frontmatter 元数据，包含 name、description、whenToUse、allowedTools、context、model、hooks 这些字段。第二部分是提示词内容，支持 $ARGUMENTS 占位符、以感叹号加反引号标记的内联 Shell 命令，以及环境变量替换。

技能解决的是重复的 AI 工作流。你让 Claude 做代码审查，每次都要写一遍"检查安全漏洞、看边界情况、注意命名规范……"。技能把这些经过验证的提示词固化下来，一次编写，反复使用。

双重调用：技能的关键创新

与传统聊天机器人的 slash command 不同，Claude Code 的技能有两条调用路径。第一是用户手动调用，比如用户输入 commit 命令，适用于用户明确需要某个流程的场景。第二是模型自动调用，由模型判断当前任务需要调用技能，例如用户说"帮我提交代码"，模型识别意图后通过 SkillTool 调用。

传统 slash command 只能手动触发——用户必须知道命令名、记住命令语法。技能的使用场景因此受限：用户不知道 review 命令存在，就永远不会用它。

双重调用让技能成为 Agent 行为的一部分。模型可以根据当前任务的上下文，判断"现在应该调用审查技能"并自动执行。用户不需要记住命令名，只需要表达意图——"帮我看看这段代码有没有问题"，模型就会选择合适的技能。

两条路径在代码层面最终汇合到相同的执行逻辑：inline 技能走 processPromptSlashCommand，fork 技能走 prepareForkedCommandContext。

技能的文件格式

每个技能是一个目录，包含一个 SKILL.md 文件。目录结构是：在 .claude/skills 目录下，每个技能占一个子目录，比如 review，里面放一个 SKILL.md 文件，内容由 frontmatter 加提示词组成。同目录下还可以有 templates 这类资源子目录，里面放 report.md 之类的模板文件。

用目录而非单文件，是因为技能可能需要附带资源文件，比如模板、配置、参考文档，并通过环境变量 CLAUDE_SKILL_DIR 引用这些资源。目录格式让技能成为一个自包含的单元。

5.2 技能来源与加载

本节回答：技能从哪里来？Claude Code 启动时做了什么？

五个来源

技能从多个来源加载，loadAllCommands 函数（位于 src/commands.ts）按顺序合并它们，findCommand 返回第一个匹配，因此排在前面的来源优先级更高。首先，第一类是内置技能，也叫 bundled 技能，通过 registerBundledSkill 在启动时注册。然后，第二类是文件系统技能，分为 managed、user、project 三级，放在 .claude/skills 目录下。之后是第三类工作流脚本、第四类插件技能，以及第五类 MCP 技能，MCP 技能来自远程服务端。最后，这些来源全部汇入同一个技能池，由 findCommand 查找。

Bundled 技能优先级最高——所以你无法通过项目技能覆盖内置技能的名称。这是有意的设计：核心技能的行为必须可预测，不能被项目配置意外替换。

文件系统技能通过 realpath 解析符号链接去重——相同规范路径的文件视为同一技能，确保在各种环境（容器、NFS、符号链接）下正确去重。

懒加载：只加载需要的

这里有一个容易被忽略但重要的设计：技能内容不在启动时加载。系统只预加载 frontmatter（name、description、whenToUse），完整的 Markdown 提示词内容在用户实际调用或模型触发时才读取。

（代码从略：src/skills/loadSkillsDir.ts 中的 estimateSkillFrontmatterTokens 函数，把技能的 name、description、whenToUse 三个字段拼接成一段文本，再用 roughTokenCountEstimation 估算这段 frontmatter 文本占用的 token 数量。）

全量加载技能内容代价不小。系统可能注册几十个技能，每个可能有几百行提示词，全部加载会挤占大量上下文空间；大部分技能在当前会话里根本用不上；全量加载还增加启动延迟，影响首次响应速度。

只加载 frontmatter，模型就知道"有哪些技能可用"；内容推迟到实际需要时再读。展示成本低，执行成本按需付。

5.3 技能发现：模型如何知道技能存在？

本节回答：技能列表如何进入模型的视野？模型如何决定何时自动触发技能？

System-reminder 注入

技能列表不是直接写在 system prompt 中的，而是作为 attachment 动态注入，最终包装成一条 system-reminder 消息。（代码从略：这条消息逐行列出当前可用的技能，每行是技能名称加一句使用时机说明。例如 update-config 用于通过 settings.json 配置 Claude Code，keybindings-help 用于自定义键盘快捷键，simplify 用于审查改动代码的复用性、质量和效率，commit 用于创建带描述信息的 git 提交。）

这个列表由 getSkillListingAttachments 函数（位于 src/utils/attachments.ts）生成。它带一个增量机制：只发送新技能，用 sentSkillNames 按 agentId 记下已发送的技能名称，避免重复注入。

System prompt 是静态的，在会话开始时确定；技能却是动态的——MCP 服务端可能在会话中途上线新技能，插件可能启用或禁用。用 attachment 承载技能列表，就能随对话推进而更新。

Token 预算：在有限空间中展示技能

技能列表需要占据上下文空间，但空间有限。formatCommandsWithinBudget 函数（位于 src/tools/SkillTool/prompt.ts）实现了一个三阶段预算分配算法。预算的计算式是：取上下文窗口 token 总数的百分之一，再按每个 token 约四个字符折算。对 200K 的上下文来说，这个预算约为 8KB。

首先，第一阶段做全量尝试，如果所有技能的完整描述加起来不超过预算，就直接使用。然后，第二阶段做分区处理：bundled 技能保留完整描述，非 bundled 技能均分剩余预算。最后，检查每个技能分到的空间是否不足 20 个字符：如果不足，就进入极端模式，非 bundled 技能仅显示名称；如果空间还够，就截断非 bundled 技能的描述来适配预算。

bundled 技能代表 Claude Code 的核心能力（commit、simplify、debug 等），用户期望这些技能始终找得到，所以永不截断。即使装了大量自定义技能、预算吃紧，核心功能的可发现性也不能牺牲——这是"核心功能优先"的取舍。

每个技能描述还有一个硬上限：MAX_LISTING_DESC_CHARS 等于 250 字符，防止单个技能的长描述挤占其他技能的空间。

whenToUse：引导模型自动触发

模型会不会自动触发技能，关键看 whenToUse 字段。它出现在技能列表中，模型据此判断"当前场景是否需要调用这个技能"。

内置技能中有一个优秀的写法模式，叫"正面触发加反面排除"。（代码从略：这个示例分两部分。正面条件是：当代码导入 anthropic、@anthropic-ai/sdk 或 claude_agent_sdk，或者用户要求使用 Claude API、Anthropic SDK、Agent SDK 时触发。反面条件是：当代码导入 openai 等其他 AI SDK，或者只是一般的编程问题时，不要触发。）

好的 whenToUse 应该做到三点。第一，描述用户意图，而非用户的措辞，"当用户需要审查代码质量时"好于"当用户说 review 时"。第二，包含否定条件，帮助模型区分相似场景，减少误触发。第三，具体而非笼统，"当用户修改了多个文件并想在提交前检查"好于"当用户需要帮助时"。

用反例来说明这些原则。（代码从略：这里列出正反两组写法。反例有三种：第一种是"当用户说 review"，描述的是措辞而不是需求，用户可能说"帮我看看代码"；第二种是"任何时候用户需要帮助"，太笼统，几乎匹配所有场景，导致频繁误触发；第三种是"当用户想用这个技能时"，属于循环定义，模型无法从中判断何时触发。正例是"当用户修改了多个文件并想在提交前检查代码质量"。）

这些触发指令是文档性的——模型根据描述自行判断，不是自动化触发器。模型可能会忽略或误判，但这是一个实用的设计：相比构建复杂的规则引擎，让模型理解自然语言描述已经足够好了。

5.4 Frontmatter 与提示词处理

本节回答：技能文件里可以写什么？提示词在执行前经过了哪些处理？

Frontmatter 字段

技能文件是 Markdown 与 YAML frontmatter。所有支持的字段分为四类。第一类是基础字段。name 是显示名称，默认使用目录名。description 是技能描述，影响模型自动触发的判断。when_to_use 是自动触发条件描述；注意唯独这个字段用下划线，其余多词字段比如 allowed-tools、argument-hint、user-invocable、disable-model-invocation 都用连字符。argument-hint 是参数提示，显示在帮助和 Tab 补全中。arguments 是命名参数列表，比如 file 和 mode，分别映射到 $file 和 $mode 占位符。第二类是执行字段。context 取 inline 或 fork，默认 inline，决定执行隔离级别。allowed-tools 是工具白名单，限制技能可使用的工具。model 是模型覆盖，取 inherit 表示继承父级。effort 是工作量级别，可以是 quick、standard 或整数。agent 指定 fork 时使用的 Agent 类型。shell 指定内联 Shell 块使用的 Shell 类型。第三类是可见性字段。paths 是 gitignore 风格的路径模式，技能仅在匹配路径下显示。user-invocable 设为 false 时，用户不可直接通过斜杠命令调用。disable-model-invocation 设为 true 时，模型不可自动触发。第四类是扩展字段：hooks 定义技能级 Hook，详见 5.8 节。

有三个字段的设计值得单独说。

paths 字段管条件可见性。parseSkillPaths 解析 gitignore 风格的路径模式，技能只在匹配路径下工作时才对模型可见。例如一个 React 组件技能可以把 paths 设置为 src/components 下的所有路径，这样在编辑后端代码时，它不会出现在技能列表中。

model 字段的 inherit 取值解析为 undefined，表示使用当前会话模型。如果主会话模型带后缀（例如 1m，表示思考预算），覆盖时会保留该后缀。

hooks 的解析通过 Zod schema 校验。无效的 hooks 定义仅记录警告但不阻止加载——一个格式错误的 hook 不应该让整个技能不可用。

提示词替换管道

技能的提示词在执行前，要先过一条多阶段预处理管道，依次是路径解析、参数绑定、环境变量注入和动态 Shell 命令执行。原始 Markdown 不会直接送给模型。每一层解决一个具体问题，层层叠加后才生成最终发送给模型的提示词。这个设计让技能既能以静态 Markdown 文件的形式定义和版本管理，又能在运行时动态适应当前项目路径、用户参数和环境上下文。

完整的替换流程由 getPromptForCommand 函数实现。首先，原始 Markdown 内容会加上基础目录前缀，在提示词开头插入一行技能所在目录的说明。然后做参数替换，处理 $ARGUMENTS、命名参数和位置参数；接着做环境变量替换，替换技能目录和会话 ID 这两个占位符。最后一步是内联 Shell 执行：如果技能来自本地，就执行提示词中嵌入的 Shell 命令块；如果来自 MCP 远程服务端，就跳过 Shell 执行。两条分支最终都产出发送给模型的最终提示词。

第一步是基础目录前缀。如果技能有关联目录（skillRoot），在提示词开头插入路径，让提示词可以引用相对路径资源。

第二步是参数替换，由 substituteArguments 函数处理多种参数格式。第一，$ARGUMENTS 替换为全部参数字符串。第二，$file 这类命名参数替换为对应的值，映射自 frontmatter 的 arguments 字段；命名参数仅支持不带花括号的写法，带花括号的 file 写法不会被识别——花括号形式仅用于内置环境变量。第三，$0 和 $1 按位置索引替换。第四，$ARGUMENTS 加方括号索引的写法，可以按索引访问参数。第五，如果提示词中没有任何占位符，参数会自动追加到提示词末尾。

第三步是环境变量替换。花括号形式的环境变量占位符会被替换：CLAUDE_SKILL_DIR 替换为技能目录路径，Windows 下反斜杠自动转为正斜杠；CLAUDE_SESSION_ID 替换为当前会话 ID。

第四步是内联 Shell 执行。技能的 Markdown 中可以嵌入以感叹号加反引号标记的 Shell 命令，执行后把输出替换回原位。例如，提示词里写"当前分支："后跟内联命令 git branch --show-current，就会变成实际的分支名；"最近提交："后跟 git log --oneline -5，就会列出最近五次提交。

所有嵌入的 Shell 命令会通过 Promise.all 并行执行，每个命令执行前都要过权限检查。MCP 技能来自远程不受信任的服务端，因此跳过 Shell 执行和环境变量 CLAUDE_SKILL_DIR 的替换——这是安全关键路径上的显式检查，详见 5.6 节。

5.5 执行模型：Inline vs Fork

本节回答：技能是如何执行的？两种执行模式有什么区别？

执行流程概览

无论用户手动输入 commit 命令，还是模型通过 SkillTool 调用，执行流程的核心路径是相同的。首先，系统接收技能调用，输入是技能名加参数，然后调用 findCommand 查找命令，找不到就返回错误。找到后检查 context 字段。如果是 fork，就走 executeForkedSkill，创建隔离子 Agent。如果是 inline，就走 processPromptSlashCommand，加载并替换提示词，再把技能内容和 contextModifier 注入对话。最后，fork 分支返回结果文本，inline 分支继续主对话。

两条入口路径的汇合是一个重要的设计细节。用户输入带提交说明的 commit 命令时，CLI 解析 slash command 语法后调用 processPromptSlashCommand。模型通过 SkillTool 调用时，SkillTool 的 call 方法最终走到的还是同一处：inline 对应 processPromptSlashCommand，fork 对应 prepareForkedCommandContext。同一个技能无论怎么触发，执行逻辑完全一致，不存在两条路径行为分叉的风险。

Inline 模式（默认）

技能的提示词作为消息注入当前对话，模型在原有上下文中继续执行。以 review 技能为例：首先，用户输入 review 命令并带上"安全性"参数。然后，CLI 调用 getPromptForCommand 做参数替换和 Shell 执行，再把技能内容作为一条对话消息注入主 Agent。最后，主 Agent 在原有上下文中执行，并把结果返回给用户。

Inline 模式的关键机制是 contextModifier。SkillTool 的 call 方法在处理完 processPromptSlashCommand 的结果后，构建一个 contextModifier 函数，它在后续回合中修改执行上下文。第一，如果技能指定了 allowedTools，就追加到 alwaysAllowRules，自动授权这些工具。第二，如果技能指定了 model，就覆盖后续回合使用的模型。第三，如果技能指定了 effort，就覆盖思考深度。

所以技能不只是注入一段提示词——它还能改变 Agent 后续的行为模式。

Inline 的优势是共享对话上下文，可以引用之前的讨论，也没有额外开销；劣势是技能的工具调用会占据主对话的上下文空间。

Fork 模式

Fork 模式创建独立的子 Agent，有自己的消息历史和工具池，完成后将结果返回父对话。以 verify 技能为例：首先，用户输入 verify 命令，CLI 调用 executeForkedSkill，创建一个拥有独立消息历史和独立工具池的子 Agent。然后，子 Agent 在隔离环境中做多轮工具调用，不影响主对话。最后，子 Agent 把结果文本返回给 CLI，CLI 再把一段"技能 verify 已完成，结果如下"的文本交给主 Agent，由主 Agent 展示给用户。

Fork 模式通过 runAgent 创建子 Agent，拥有完全隔离的上下文。子 Agent 完成后，clearInvokedSkillsForAgent 会按 agentId 清理其技能记录，防止状态泄漏。

Fork 的优势是不污染主对话上下文、能限制工具集做安全隔离、还能换用不同模型；劣势是不能引用主对话历史，创建子 Agent 也有额外开销。

对比与选择

两种模式可以从五个维度对比。第一，对话历史方面，inline 共享主对话，fork 独立隔离。第二，工具池方面，inline 使用主 Agent 的全部工具，fork 受 allowedTools 限制。第三，上下文影响方面，inline 占据主上下文空间，fork 不影响主上下文。第四，模型方面，inline 默认当前模型但可覆盖，fork 可指定不同模型。第五，结果形式方面，inline 直接在对话中输出，fork 汇总为一段文本返回。

选择 Fork 的场景有四种。第一，需要大量工具调用时，比如运行完整测试套件，避免污染主上下文。第二，需要限制可用工具时，比如审查技能不应写文件，实现权限隔离。第三，需要使用更便宜的模型做快速检查，实现成本优化。第四，需要失败隔离，fork 失败不影响主对话流。

实战示例

第一个例子是代码审查，用 Fork 模式加只读工具。（代码从略：这是一个技能文件示例，frontmatter 中 description 写的是审查当前分支的所有改动，when_to_use 写的是当用户要求审查代码质量时，allowed-tools 限制为 Bash、Read、Grep、Glob 四个只读工具，context 设为 fork。提示词正文要求审查当前分支相对于 main 的所有改动，并用 $ARGUMENTS 接收用户指定的关注点。）

审查需要大量 git diff、Read、Grep 调用，放在主对话里会污染上下文，所以走 fork。allowed-tools 限制为只读——审查不应修改代码。

第二个例子是代码风格修复，用 Inline 模式。（代码从略：这是一个技能文件示例，frontmatter 中 description 写的是检查并修复最近修改文件的代码风格，when_to_use 写的是当用户修改了代码后想检查风格一致性时，没有配置其他执行字段。提示词正文要求检查最近修改的文件是否符合项目代码风格，如果有问题就直接修复。）

修复代码要用 Edit 工具，需要完整的工具权限，所以走 inline。工具调用量不大，占不了多少主对话上下文。

第三个例子是快速扫描，用 Fork 模式加轻量模型。（代码从略：这是一个技能文件示例，frontmatter 中 description 写的是快速检查代码的明显问题，context 设为 fork，model 指定为 claude-sonnet，effort 设为 quick，allowed-tools 限制为 Read、Grep、Glob。提示词正文要求快速扫描用户指定文件的明显问题，重点是未处理的异常、硬编码密钥和明显的逻辑错误。）

用 Sonnet 更快更便宜，fork 隔离，effort 的 quick 进一步降低思考深度。三个维度的资源优化叠加。

5.6 安全与信任模型

本节回答：技能如何确保安全？不同来源的技能受到什么程度的限制？

信任层级

不同来源的技能有不同的信任级别，安全限制随信任度降低而增加。第一，managed 来源即企业策略技能，信任级别最高，由企业管理员审核过，完全信任。第二，bundled 内置技能，信任级别高，由 Claude Code 团队维护。第三，project 和 user 技能，信任级别中等，安全属性自动允许，其他需确认。第四，plugin 插件技能，信任级别中低，属于第三方代码，需要启用的显式同意。第五，MCP 技能，信任级别最低，来自远程不受信任的服务端，被禁用 Shell 执行和路径暴露。

SAFE_SKILL_PROPERTIES：前向兼容的权限设计

SkillTool 在执行技能前检查权限。这里有一个关键优化：只包含"安全属性"的技能自动允许，无需用户确认。

"安全属性"由 SAFE_SKILL_PROPERTIES 白名单定义（位于 src/tools/SkillTool/SkillTool.ts）。skillHasOnlySafeProperties 函数遍历技能对象的所有键，检查是否全部在白名单中。

白名单和黑名单的差别，在新增属性时才暴露。假设未来 PromptCommand 类型增加了一个 networkAccess 属性。白名单模式下，networkAccess 不在白名单中，默认就要走权限审批，是安全的；黑名单模式下，它没被加进黑名单，默认放行，这就是安全漏洞。

白名单的代价只是"遗漏时多一次用户确认"，黑名单的代价是"遗漏时出现安全漏洞"。在安全敏感场景中，默认拒绝比默认允许更安全，因为遗忘的后果是不对等的。

MCP 技能的安全隔离

MCP 技能来自远程服务端，按不受信任的代码对待，限制也最严格。（代码从略：这段代码位于 src/skills/loadSkillsDir.ts 的 getPromptForCommand 内，逻辑是只有当技能不是从 MCP 加载时，才调用 executeShellCommandsInPrompt 执行提示词中的内联 Shell 命令；注释写明这是安全考虑，因为 MCP 技能是远程且不受信任的。）

限制有两条。第一，内联 Shell 一律不执行，远程提示词里嵌入的强制删除文件之类的命令不会运行。第二，环境变量 CLAUDE_SKILL_DIR 也不做替换——它对远程技能无意义，且暴露本地路径是信息泄露。

注意这个检查是显式实现的——直接在代码中写"如果加载来源不是 MCP"这样的判断，而非依赖某个抽象层的过滤。安全关键路径上的显式检查比隐式依赖更可靠，因为你可以直接看到"什么被阻止了"。

Fork 模式的安全意义

Fork 不只是"在另一个线程运行"——它提供了三重隔离。第一，权限隔离：allowedTools 限制子 Agent 可用的工具。审查技能把 allowed-tools 设置为 Bash、Read、Grep、Glob，即使提示词被注入恶意指令，也无法写文件。第二，上下文隔离：子 Agent 看不到主对话历史，也不会向主对话泄露信息。第三，模型隔离：可以用不同模型，比如用 Sonnet 做快速检查而非 Opus。

内置技能的安全文件提取

部分 bundled 技能需要在运行时提取资源文件到磁盘。safeWriteFile 使用了多重安全措施防止攻击。第一，使用 O_NOFOLLOW 加 O_EXCL 标志防止符号链接攻击——攻击者可能预先在目标路径创建指向敏感文件的符号链接。第二，路径遍历检查：resolveSkillFilePath 拒绝包含上级目录引用或绝对路径的文件名。第三，文件权限设为仅属主可读写，即 0o700 或 0o600，只有当前用户可读写。第四，懒提取加 memoize：extractionPromise 确保多个并发调用等待同一个提取完成，而不是竞争写入。

5.7 长会话中的技能持久性

本节回答：对话压缩后，技能指令会丢失吗？

问题

当对话过长触发 autocompact（上下文压缩）时，之前注入的技能提示词会被压缩摘要覆盖。模型失去对技能指令的访问——压缩前按技能指令行事，压缩后"忘记"了技能。

如果不解决这个问题，一个长时间的编码会话会逐渐"衰减"：第 50 轮使用 commit 命令的行为，可能与第 5 轮不一致。

解决方案

addInvokedSkill 在每次技能调用时记录完整信息到全局状态，并按 agentId 隔离。（代码从略：这段代码位于 src/bootstrap/state.ts，addInvokedSkill 记录技能的名称、路径、完整内容、时间戳和所属 Agent ID。）

压缩后，createSkillAttachmentIfNeeded 从全局状态重建技能内容，作为 attachment 重新注入。

预算管理

恢复带明确的预算上限。（代码从略：这里定义了两个预算常量，压缩后的技能恢复总预算是 25000 个 token，单个技能的上限是 5000 个 token。）

分配策略有三条。第一，按 invokedAt 时间戳排序，最近调用的优先，也就是按调用时间从近到远排，因为最近使用的技能最可能仍然相关。第二，超出单技能上限时，保留头部、截断尾部，因为技能的设置指令和使用说明通常在开头。第三，超出总预算时，最不活跃的技能被丢弃。

Agent 作用域隔离

记录的技能按 agentId 隔离——子 Agent 调用的技能不会泄漏到父 Agent 的压缩恢复中，反之亦然。clearInvokedSkillsForAgent 在 fork Agent 完成时清理其技能记录。这确保了压缩恢复的精确性：每个 Agent 只恢复自己实际使用过的技能。

5.8 扩展机制与设计洞察

本节回答：技能系统如何支持扩展？整体设计有哪些值得学习的地方？

技能级 Hook

技能可以在 frontmatter 中定义自己的 Hook，在技能执行期间生效。（代码从略：这段 YAML 在 frontmatter 中定义了一个 PreToolUse 类型的技能级 Hook，它匹配 Bash 工具的调用，并执行 validate-deploy-command.sh 这个校验脚本。）

技能级 Hook 不覆盖全局 Hook（settings.json），而是与之叠加：两者同时生效，全局 Hook 先执行。企业管理员设置的安全 Hook 因此不会被技能绕过。

注册时机也遵循懒加载原则：技能的 Hook 在调用时才注册，由 registerSkillHooks 完成。校验通过 Zod schema 完成，格式错误的 Hook 记录警告但不阻止技能加载——容错优先。

内置技能架构

内置技能通过 registerBundledSkill 函数（位于 src/skills/bundledSkills.ts）在启动时注册，内容编译在二进制中，不需要运行时文件读取。（代码从略：这段代码调用 registerBundledSkill 注册一个名为 simplify 的内置技能，提供名称和描述，标记 userInvocable 为 true，并通过异步的 getPromptForCommand 返回预编译的提示词；如果带了用户参数，就在提示词后面附加一段"额外关注点"。）

需要引用资源文件的技能通过 files 属性声明，首次调用时提取到临时目录。具体路径是临时目录下的 bundled-skills 文件夹，里面依次是版本号、每进程随机的 nonce 和技能名，而不是放在用户主目录的 .claude 下。这个路径按版本号隔离旧二进制的残留，再加随机 nonce 防越权访问；extractionPromise 做了 memoize，保证并发安全。技能提示词会自动添加"Base directory for this skill"加目录路径的前缀。

内置技能的可用性有两套不同机制。第一是注册期门控，用 feature 函数判断是否注册，比如 claudeApi 技能，只在 BUILDING_CLAUDE_APPS 特性为真时才注册，条件不满足则根本不注册。第二是注册后由 isEnabled 回调动态判断可见性，技能无条件注册，但每次是否展示由回调决定。比如 loop 技能由 isKairosCronEnabled 回调判断，还有 keybindings 和 claude-in-chrome。

设计洞察

第一，发现与执行分离。Frontmatter 用于浏览和发现，成本低；完整内容只在执行时按需加载。这是管理大型工具集的通用模式——展示目录不需要加载全部内容，对上下文空间宝贵的 AI 系统尤为重要。

第二，白名单权限是前向兼容的安全。新增属性默认需要权限审批：遗漏的黑名单条目是安全漏洞，遗漏的白名单条目只是多一次用户确认。这种不对称性决定了白名单是更安全的选择。

第三，双重调用扩展了技能的适用范围，让技能从"用户必须记住的命令"变为"Agent 自动选择的能力"。用户表达意图，Agent 选择工具——这更接近人类协作的模式。

第四，Fork 模式同时提供权限隔离、上下文隔离和模型隔离。三重隔离让 fork 既是性能优化，也是安全边界；设计安全敏感的技能时，fork 应该是默认选择。

第五，压缩后恢复确保长会话一致性。它解决的是一个容易被忽略的问题——随着对话增长，技能指令会被压缩掉。按时间优先的预算分配是一个实用的启发式：最近使用的技能最可能仍然相关。

动手实践：在 .claude/skills 目录下创建一个自定义技能。从最简单的 inline 技能开始——只需要一个技能名目录下的 SKILL.md 文件。观察它如何出现在斜杠补全列表中，以及模型如何根据 when_to_use 自动触发它。

上一章：工具系统。下一章：记忆系统。
