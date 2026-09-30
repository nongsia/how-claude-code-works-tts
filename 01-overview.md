---
title: 第 1 章：概述（朗读版）
---

第 1 章：概述

本文是 docs/01-overview.md 的朗读版，表格与代码已转为口语描述，内容未增删。

本章导读：本章从宏观视角介绍 Claude Code 的定位、技术栈和架构全貌：1.1 节讲它作为 Agent 与传统编程辅助工具的区别，1.2 节讲技术选型，1.3 节给出 6 条核心设计原则，1.3 节后面是关键术语速查，1.4 节是源码目录结构，最后由 1.5 节数据流全景、1.6 节启动流程、1.7 节架构总览把所有概念串联起来。如果你只想快速了解全貌，可以直接跳到 1.5 节数据流全景。

1.1 Claude Code 解决什么问题

Claude Code 不是一个简单的"CLI 调用大模型"工具。它是 Anthropic 官方推出的受控工具循环 Agent（Controlled Tool-Loop Agent），专为真实软件工程任务设计。

从工具到 Agent：三级范式

Claude Code 的定位，要放在 AI 辅助编程的三级范式里看。

第一级是代码补全，比如 Copilot。模型的工作是"预测下一行代码"。它看到你的光标位置和上下文，生成一个补全建议。这是一个单次预测问题——模型不需要理解整个项目，不需要执行任何操作，只需要根据局部上下文生成合理的代码片段。用户始终是驾驶员，模型只是副驾。

第二级是 IDE 聊天助手，比如 Cursor Chat 和 Copilot Chat。用户可以用自然语言描述需求，模型生成代码片段或修改建议。这比补全强大——模型可以看到更多上下文，可以生成多个文件的修改。但有一条关键限制：模型不能执行操作。它生成一个 diff，由用户决定是否 apply。如果 diff 有问题，比如依赖了一个不存在的函数，用户需要手动发现并反馈，模型无法自行验证。

第三级是自主 Agent，也就是 Claude Code。模型不仅生成代码，还能自主执行多步操作。考虑一个真实场景：你想给项目添加一个新的 REST endpoint。Copilot 会给你一个函数体。IDE 聊天可能会建议一个修改方案。而 Claude Code 的做法是：先用 Grep 搜索现有路由定义理解项目的路由模式，用 FileRead 读取中间件配置，然后创建 handler 文件、注册路由、编写测试，接着运行 npm test 发现测试失败，读取错误信息，修复代码，再次运行测试直到通过，最后提交 git commit。整个过程是一个自主决策循环——模型决定下一步做什么，执行后观察结果，再决定下一步。

这种范式跃迁带来了根本性的架构差异。一个 Agent 需要四样东西：第一是循环，即 Loop，反复地思考、执行、观察，直到任务完成才停。第二是工具，即 Tools，能读文件、写文件、执行命令，而不只是生成文本。第三是记忆，即 Memory，记住用户偏好和项目上下文，不必每次从零开始。第四是安全控制，即 Safety，在用户机器上执行真实操作，需要严格的权限管理。

Claude Code 的每一个架构决策都围绕这四个需求展开。

与其他 Agent 的区别

市面上不乏其他编程 Agent，比如 AutoGPT、OpenDevin、Aider 等，但 Claude Code 有一个独特优势：它由构建 Claude 模型的同一个团队开发。这意味着系统提示词、工具描述、错误处理策略都与模型的行为特性共同设计和调优。例如，Claude Code 的 system prompt 不是一个通用的"你是一个编程助手"——它包含了针对 Claude 模型特性优化的详细行为指令，工具的 description 字段也经过反复调优以匹配模型的理解模式。

此外，Claude Code 在生产级工程质量上远超大多数开源 Agent 项目：7 层纵深防御的安全系统、4 级渐进式上下文压缩、7 种错误恢复策略、流式工具预执行——都是服务真实用户的工业级实现，不是学术 demo。

"Agent-first" 的架构含义

"Agent-first" 不是营销口号——它有具体的架构含义：模型是循环中的决策者，而非人类。人类做两件事：设定目标，比如"给这个项目添加用户认证"；审批危险操作，比如"确认执行 npm install？"。两次人类交互之间，模型自主决定读什么文件、改什么代码、执行什么命令。

这体现在源码中最核心的一行，也就是 src/query.ts 第 307 行的 while (true)。这一行循环体里反复做的事情是：压缩，然后 API 调用，然后工具执行，最后继续或者退出。

这个循环只有当模型的响应不包含任何工具调用时才会退出。换句话说，是模型——而不是代码逻辑——决定任务是否完成。代码只是提供了执行环境，真正的"大脑"是模型本身。

1.2 技术栈

技术选型分为七层。第一层是运行时，选用 Bun，这是一个高性能 JS/TS 运行时，支持编译时 Feature Flag 消除。第二层是语言，全量使用 TypeScript，严格类型检查。第三层是 UI 框架，采用 React 加自研的 Ink 渲染器，这是一个基于 React 的终端 UI 框架，渲染器位于 src/ink 目录，约 1.0MB。第四层是布局引擎，选用 Yoga，这是 Facebook 的 Flexbox 布局引擎，适配终端。第五层是 Schema 验证，选用 Zod，做运行时类型校验，用于工具输入、Hook 输出和配置验证。第六层是 CLI 框架，选用 Commander.js，负责命令行参数解析，分发到 REPL、headless 和 SDK 三种模式。第七层是 API 协议，选用 Anthropic 官方 TypeScript SDK，支持流式响应。

技术选型本身不是本文重点，但有两个选择深刻影响了架构设计，值得单独说。第一个是 Bun 的 feature() 宏。Claude Code 内部有大量功能，比如协调器模式、Swarm 团队等，在外部发布版本中需要完全移除。Bun 提供的编译时 Feature Flag 让这些代码在构建时被物理删除，而非运行时隐藏。这在后续"编译时 Feature Gate"设计原则中会详细展开。第二个是自研 React 终端渲染器。Claude Code 的终端 UI 复杂度远超普通 CLI——权限确认对话框、流式代码高亮、嵌套工具进度指示器都需要组件化的状态管理。团队维护了一个约 1.0MB 的定制 Ink 渲染器，而非使用上游库，详见第 14 章：用户体验设计。

1.3 核心设计原则

Claude Code 的架构遵循 6 条核心设计原则。

1、Generator-based 流式架构

从 API 调用到 UI 渲染，全链路使用 async function 加星号的异步生成器，串成一条流式处理管道，每个 Token、每个工具结果都能实时流向用户界面。

核心查询循环的签名如下。（代码从略：这段代码定义在 src/query.ts，是 query 函数的类型签名。它是一个异步生成器，依次产出 StreamEvent、Message、ToolUseSummaryMessage 等类型的流式事件，返回类型中的 Terminal 代表查询的最终状态。）

这是一个异步生成器——边执行边 yield 事件，不必等全部完成再返回。调用方通过 for await 循环实时消费 query(params) 吐出的每一个事件，交互式的调用方是 REPL，无头和 SDK 的调用方是 QueryEngine。模型输出的每个 Token、每个工具调用的结果、压缩事件、错误恢复——所有这些都通过同一个 generator 管道流向 UI 层。

这种设计的好处是零缓冲延迟：用户在模型开始生成的瞬间就能看到输出，不需要等整个响应完成。

为什么是 Generator 而不是 Callback 或 Promise？三种异步模式各有根本性的局限。第一是 Callback 模式，即经典 Node.js 风格，容易陷入 callback hell；更重要的是无法优雅地传递 backpressure，当 UI 渲染跟不上数据产生速度时，没有自然的暂停机制。当用户按 Ctrl+C 中断时，需要手动在每一层 callback 中接线取消逻辑。第二是 Promise 和 async-await 模式，它解决了 callback hell，但 await 是阻塞式的——一次 await apiCall() 必须等到整个响应完成才能返回。要实现流式，你需要手动缓冲部分结果并轮询，这等于在 Promise 之上重新发明 generator。第三是 Generator 模式，yield 天然就是流式语义——API 层作为生产者，产出一个 token 就 yield 一次；UI 层作为消费者，按自己的节奏拉取。更关键的是，generator.return() 可以级联清理整个调用链：用户按下 Ctrl+C，REPL 调用 generator.return()，QueryEngine 的 generator 随之终止，query() 的 generator 跟着终止，API 请求最终被 abort。不需要手动接线，cleanup 沿着 generator 链自动传播。

注意 query() 的返回类型 AsyncGenerator——其中 Terminal 是 generator 的 return type，代表查询的最终状态，与 yield 出的中间事件流是分离的。这种双通道设计，一边 yield 流式事件，一边 return 最终结果，只有 generator 能干净地表达。

整个数据流形成嵌套的 generator 管道，入口分两条。交互式路径由 REPL 直接调用 query()：REPL.tsx 第 146 行 import 了 query，主循环在 REPL.tsx 第 2793 行，用 for await 循环逐个消费 query 产出的事件。无头、-p 和 SDK 路径先经会话引擎再汇入核心循环：cli/print.ts 先调 QueryEngine 的 submitMessage() 或 ask()，再进入 query()。两条路径最终都收敛到同一个 query()，由它调用 queryModelWithStreaming()，后者位于 services/api/claude.ts。每一层 generator 在管道上叠加自己的处理逻辑——压缩、错误恢复、权限检查——但对上层来说，它只是一个统一的 AsyncGenerator 事件流。

2、防御性分层安全

以攻击面最大的 Bash 命令为例，权限系统的核心决策路径是一条多层流水线。这条流水线自上而下是：首先做权限规则匹配，位于 src/utils/permissions 目录；通过后做 Bash AST 分析，位于 src/utils/bash 目录，用 tree-sitter 解析；接着是 23 项静态安全验证器；再通过后交给 LLM 分类器，也就是 yoloClassifier；最后是用户确认对话框。每一层通过后才进入下一层。

这条路径只是 Bash 命令核心决策路径的一个代表性切片。完整的权限系统共 7 层——在上述环节之外还有工作区信任确认、权限模式、沙箱与隔离等——完整模型详见第 12 章。

为什么需要这么多层？理解威胁模型是关键：Claude Code 在用户的真实机器上执行任意代码。模型不是完美的——它可能因为上下文混淆而生成错误命令，可能被恶意 README 中的 prompt injection 误导，或者只是单纯犯了一个逻辑错误。一条 rm -rf 这样的删除命令就足以造成不可挽回的损失。

这是经典的纵深防御：即使某一层有 bug 或被绕过，其他层仍然可以阻止危险操作。每一层使用不同的技术手段，覆盖不同类别的风险。

第一层是权限规则匹配，位于 src/utils/permissions 目录，由 permissions.ts 的 hasPermissionsToUseTool 做 allow 或 deny 判定。这是策略层——用户通过 CLAUDE.md 的 allowedTools 配置或 --allowedTools 标志声明哪些操作是被允许的。这一层表达的是用户意图：在这个项目中，运行 npm test 总是安全的。这层之上的 src/hooks/toolPermission 目录只是按执行场景分派权限请求、编排确认对话框的胶水层，本身不是规则引擎。

第二层是 Bash AST 分析，位于 src/utils/bash 目录，基于 tree-sitter。它不是用正则匹配命令字符串，而是用 tree-sitter 将 Bash 命令解析为抽象语法树。为什么不用正则？因为 Bash 语法极其灵活——把 rm 拆开加引号、用命令替换间接生成、再用 eval 包装，这些变形都能绕过简单的字符串匹配，但 AST 分析能识别出实际执行的命令。

第三层是 23 项静态安全验证器，做硬编码的已知危险模式检查。这是白名单和黑名单层——某些操作，比如写入 /etc/passwd、修改 SSH 配置，无论上下文如何都应该被拦截。

第四层是 LLM 分类器，也就是 yoloClassifier。它通过 sideQuery 向 Claude 模型发起带专用 system prompt 的旁路请求，对命令做 stage1 和 stage2 两阶段安全判定——名为分类器，其实没有单独训练的模型，复用的是主模型的旁路推理。它捕获的是静态规则覆盖不到的新型危险模式，比如一条看起来无害但在当前上下文中可能造成问题的命令。

第五层是用户确认对话框，也就是最终的人类审核。即使所有自动化层都放行了，用户仍然可以看到即将执行的操作并选择拒绝。

这套分层的要点是各层使用完全不同的技术——规则匹配、语法解析、LLM 判定、人类判断——单一类别的 bug 因此无法同时绕过所有层。即使权限规则配置错误地放行了一条命令，tree-sitter AST 分析仍然会检测到 rm -rf 根目录这样的结构性危险模式。

3、编译时 Feature Gate

通过 Bun bundler 的 feature() 宏实现编译时死代码消除。内部功能（如协调器模式）在外部构建中完全移除——删除发生在编译期，发布产物里根本不存在这些代码。

这个模式在整个代码库中反复出现。（代码从略：这段代码位于 src/query.ts 开头，是 6 个 Feature Gate 的条件加载。它对 REACTIVE_COMPACT、CONTEXT_COLLAPSE、EXPERIMENTAL_SKILL_SEARCH、TEMPLATES、HISTORY_SNIP、BG_SESSIONS 六个特性名逐一调用 feature() 判断，开启时才加载对应模块，否则置为 null。）

as typeof import 这样的类型断言让 TypeScript 在编译期获得正确的类型信息，而 feature() 在 Bun bundler 构建时被求值——如果结果为 false，整个 require 分支和相关代码都被 tree-shaken 移除。使用这些模块的代码总是先用 if 判断模块是否存在，这个条件判断本身也在编译时被消除。

4、状态集中加不可变更新

全局状态集中于 bootstrap/state.ts，共 1758 行，有 150 多个访问器。为什么不直接用全局变量？

Claude Code 有约 55 个工具，这是工具注册表的口径，含受 feature 开关控制的工具，快照可见约 40 个 buildTool 定义；此外还有多个子 Agent、压缩管道和 React UI。这样的系统里共享状态不可避免——当前使用的模型名、会话 ID、Feature Flag 缓存、累计成本、文件修改状态等，都需要被多个子系统同时访问和修改。朴素的全局变量方案会带来三个实际问题：第一是 import 循环，模块 A 导入 B 的状态，B 导入 C 的工具，C 又导入 A 的状态——在一个 1900 文件的项目中，这种循环几乎不可避免。第二是不可追踪的修改，当某个 bug 导致模型名被意外改变，你无法设断点查看是谁在什么时候改了这个值。第三是 React 渲染问题，直接修改全局对象的属性不会触发 React 组件的重新渲染。

bootstrap/state.ts 的解决方案是通过显式的 getter 和 setter 函数暴露状态，比如 getSessionId()、getTotalCostUSD()、setMainLoopModelOverride()，而不是导出可变对象。每个模块只导入自己需要的 getter 和 setter 函数，从而打破 import 循环；每次修改都经过函数调用，可以轻松添加日志或断点追踪。

UI 状态使用 Zustand 模式的不可变更新——setAppState 接收一个函数，在旧状态的基础上展开并叠加新字段——保证 React 组件能正确感知状态变化。

这不是一个"理想"的架构——团队自己也在控制全局状态的增长。但在 Claude Code 这样的复杂系统中，集中管理的 getter 和 setter 是一个务实的平衡：比全局变量安全，比完整的状态管理框架（如 Redux）轻量。

5、渐进式压缩

压缩流水线分四级：Snip、Microcompact、Context Collapse、Autocompact，确保对话永不因上下文溢出而中断。四级按成本从低到高排列，每级解决不同粒度的问题。

第一级是 Snip，零 API 成本。它移除对话历史中已经不再被引用的旧工具结果。例如，10 轮前的一次 grep 搜索结果可能有 50KB，但模型早已不再关注它。Snip 用一个占位符替换这些内容，纯本地操作，不需要调用 API。对应 query.ts 第 401 到 410 行。

第二级是 Microcompact，近零成本。它压缩单个工具结果的体积。比如一个 Grep 工具返回了 200 行匹配结果，Microcompact 可以将其截断为最相关的前 20 行。同样是本地启发式操作。对应 query.ts 第 414 到 426 行。

第三级是 Context Collapse，中等成本。它将相关的消息序列分组折叠为摘要。关键设计是，这是一个读时投影——原始完整历史保留在内存中，发送给 API 的是折叠后的视图。这意味着折叠是可逆的，不会丢失原始信息。对应 query.ts 第 440 到 447 行。

第四级是 Autocompact，全量成本。它 fork 一个子 Agent 生成整个对话的摘要，用摘要替换原始历史。这是"核选项"——释放最多空间，但不可逆地丢失对话细节。对应 query.ts 第 454 到 467 行。

为什么四级而非只用 Autocompact？如果只有 Autocompact，每次上下文接近满就必须调用 API 生成摘要——用户要多等一次，细节也随摘要丢失。先执行零成本的 Snip 和 Microcompact，系统往往能释放足够的空间，避免触发昂贵的 Autocompact。实践中，很多对话自始至终都不需要走到 Autocompact 这一步。

详见第 3 章：上下文工程。

6、工具即扩展点

所有能力——文件操作、搜索、Agent 派生、MCP 桥接——统一为 Tool 接口，定义在 src/Tool.ts。无论是内置的 BashTool、通过 MCP 协议接入的外部工具，还是插件系统注册的第三方工具，它们共享完全相同的执行管道：先过权限检查，再做输入校验，然后执行，结果经格式化后交给 UI 渲染。

Tool 接口拥有约 20 个字段和方法，每一个都在统一管道中扮演角色。第一个是 isReadOnly()，告诉权限系统这个工具是否只读——只读工具，如 Grep、Glob，可以跳过用户确认。第二个是 isConcurrencySafe()，告诉 StreamingToolExecutor 这个工具能否与其他工具并行执行——Grep 可以，但 FileEdit 不行，因为可能产生写冲突。第三个是 shouldDefer，告诉 API 层是否延迟发送完整 schema——数十个工具的 schema 加起来占用大量 token，不常用的工具可以按需加载。第四个是 inputSchema，基于 Zod，模型生成的参数在执行前必须通过 Schema 验证，防止畸形输入触达工具执行层。第五个是 interruptBehavior()，定义用户中断时工具的行为——有些工具可以立即中断，有些需要清理。

findToolByName() 函数不区分工具来源——对 query 循环来说，所有工具都是平等的 Tool 对象。这意味着一个通过 MCP 协议接入的外部 Kubernetes 工具，和内置的 BashTool 经历完全相同的权限检查、输入验证、结果格式化流程。扩展 Claude Code 的能力就是实现一个符合 Tool 接口的对象，而不需要修改核心循环——这是经典的开闭原则（Open-Closed Principle）在 Agent 架构中的体现。

核心术语速查

在深入后续章节之前，以下是贯穿整个代码库的几个核心概念。第一个是 State，指单次 query 循环的可变状态对象，包含消息历史、压缩追踪、output token 恢复计数等，对应 query.ts 第 204 行的 type State 定义。第二个是 Tool，统一的工具接口，所有能力，包括内置、MCP 和插件，都实现此接口，共享同一执行管道，对应 Tool.ts。第三个是 Message，对话中的一条消息，包含 UserMessage、AssistantMessage、ToolUseSummaryMessage 等子类型，对应 types/message.ts。第四个是 StreamEvent，generator 管道中 yield 出的事件单元，代表一个 token、工具结果或状态变更，同样对应 types/message.ts。第五个是 Terminal 和 Continue，这是 query 循环的两种转移状态——Terminal 表示循环结束，Continue 表示需要继续下一轮迭代，对应 query/transitions.ts。第六个是 QueryEngine，会话级引擎，管理对话生命周期，包括持久化、预算和结果组装，是无头和 SDK 入口与核心循环之间的边界，交互式 REPL 不经它而直接调用 query()，对应 QueryEngine.ts。

1.4 源码目录结构

Claude Code 源码约 1900 个文件，TypeScript 代码 512K 行以上，目录结构如下。根目录下是几个核心文件：main.tsx 是 CLI 主入口，共 4683 行，由 Commander.js 解析参数，分发到 REPL、headless 和 SDK 三种模式。QueryEngine.ts 是会话引擎，共 1295 行，管理对话全生命周期，包括消息持久化、预算追踪和结果组装。query.ts 是核心查询循环，共 1729 行，是单次查询的状态机，串起压缩、API 调用、工具执行和恢复继续。Tool.ts 是所有工具的统一类型约束，tools.ts 负责工具注册与组装，context.ts 负责上下文构建，提供 Git 状态、CLAUDE.md 和日期等信息。

主要子目录方面，bootstrap 存放集中式全局状态，其中的 state.ts 共 1758 行，提供 150 多个 getter 和 setter，所有子系统通过访问器读写共享状态，避免 import 循环。entrypoints 是入口点，init.ts 共 340 行，做 14 步幂等初始化，cli.tsx 处理 --version、MCP server 和 bridge 等快速路径，sdk 是 SDK 入口与类型。screens 是主要界面，REPL.tsx 是 875KB 的主对话界面，负责消息渲染、输入处理和状态管理，另有诊断界面 Doctor.tsx 和恢复对话的 ResumeConversation.tsx。

tools 目录是内置工具，注册表含 feature 开关的约 55 个，快照可见约 40 个 buildTool 定义。代表性的有执行 Shell 命令的 BashTool、派生子 Agent 的 AgentTool、读文件的 FileReadTool、编辑文件的 FileEditTool、基于 ripgrep 的 GrepTool、匹配文件的 GlobTool、获取网页的 WebFetchTool 和调用技能的 SkillTool 等。

services 目录里，api 子目录是 API 客户端层，其中 claude.ts 共 3419 行，是 HTTP 到 Claude API 的桥梁，负责 prompt 构建、缓存控制、thinking 配置、task budget 注入和流式响应解析；另有重试策略 withRetry.ts 和缓存断裂检测。compact 子目录是压缩系统，autoCompact.ts 负责自动压缩触发，compact.ts 共 1705 行，fork 子 Agent 生成对话摘要。此外还有 MCP 协议集成，支持 6 种传输和 8 种服务端配置类型，以及 OAuth 2.0 加 PKCE、插件系统和语言服务器协议。

hooks 目录是 React 自定义逻辑所在的 UI 层，其中 toolPermission 子目录负责工具权限请求的分场景处理与对话框编排，按执行场景分为交互确认、coordinator 和 swarmWorker 三个权限处理器。

其余目录中，coordinator 是多 Agent 协调器，属于内部功能，由 feature 开关控制；memdir 是记忆系统，按项目存放在用户主目录下 .claude 的项目记忆目录中；skills 是技能系统，内置 14 个技能，通过 registerBundledSkill 注册；ink 是约 1.0MB 的自定义终端渲染器，把 React 输出转换成终端显示；还有 vim 模式、存放 Zod Schema 定义的 schemas，以及 utils 通用工具库，utils 里有 Hook 执行引擎、Bash AST 解析、共 5512 行的消息处理 messages.ts，以及 Token 估算与追踪。

1.5 数据流全景

理解 Claude Code，关键是看清数据如何在各层之间流动。下面是一次完整的用户交互的数据流。首先，用户在终端输入消息，进入 REPL 组件，斜杠命令和附件在这里处理，普通消息则交给核心循环。交互式路径由 REPL 直接调用 query()；无头、-p 和 SDK 路径先经过 QueryEngine，再汇入同一个核心循环。然后进入反复执行的查询循环，直到模型不再调用工具：每一轮先运行 4 级压缩流水线，再调用 callModel()，发送系统提示、消息和工具列表。模型响应按 token 逐个流式返回，作为 StreamEvent 事件实时渲染到界面。如果模型调用了工具，StreamingToolExecutor 会在流式过程中立即执行，不等流结束；结果收集完毕后注入对话历史，继续下一轮循环。最后，当响应不含工具调用时，query() 返回 Terminal，无头和 SDK 路径经 QueryEngine 组装结果，界面显示最终响应。

沿着数据流逐步展开，看每个阶段发生了什么。

第 1 步，用户输入进入 REPL。React 组件 REPL.tsx 捕获用户的文本输入。如果是斜杠命令，命令文本本身不会作为用户消息发给模型；像 /clear、/help 这样的纯本地命令完全在本地处理；/compact 这类命令虽然也在本地处理，过程中却会自行发起一次 API 调用，fork 子查询生成摘要。普通消息则进入核心循环——交互式由 REPL 直接调用 query()，无头、-p 和 SDK 则先经 QueryEngine 的 submitMessage()。

第 2 步，QueryEngine 准备查询。processUserInput() 处理消息中的附件，包括图片缩放和文件引用解析，构建包含消息历史、系统提示词、工具列表和权限上下文的 QueryParams 对象，然后调用 query() 启动核心循环。

第 3 步，4 级压缩管道运行。注意——压缩不是只在对话开始时运行一次，而是在每次 API 调用之前都会运行。循环的每一轮迭代都会依次检查 Snip、Microcompact、Context Collapse、Autocompact，按需触发。大多数迭代中没有任何压缩触发，因为上下文还没满；但当历史消息累积到接近上下文窗口上限时，压缩管道会自动介入。

第 4 步，API 调用。系统提示词、压缩后的消息历史和工具 schema 被发送到 Claude API，通过 services/api/claude.ts 的 queryModelWithStreaming()。响应以 token-by-token 的方式流式返回，每个 token 被 yield 为 StreamEvent，沿着 generator 链向上传递到 UI，用户立即看到文字出现。

第 5 步，工具在流式过程中即开始执行。这是 Claude Code 的一个重要性能优化：StreamingToolExecutor 不等待模型的完整响应。当流式解析器检测到一个 tool_use JSON block 已经完整，工具执行立即开始——此时模型可能还在继续生成后面的文字或其他工具调用。只读且并发安全的工具，如 Grep、Glob，甚至可以并行执行，进一步缩短多工具调用的总耗时。

第 6 步，结果注入，循环继续。工具的执行结果被封装为 tool_result 消息追加到对话历史中。循环回到第 3 步——再次检查是否需要压缩，再次调用 API。模型看到工具结果后决定下一步：继续调用更多工具，或者生成最终的文字回复。

第 7 步，循环退出，结果组装。当模型的响应中不包含任何 tool_use block 时，query() 返回一个 Terminal 值。QueryEngine 组装最终结果，持久化对话历史，更新 usage 和 cost 追踪。

关于性能：流式工具预执行

第 5 步中的"流式工具预执行"值得单独强调。在朴素的实现中，流程是串行的：等模型响应完整返回，解析出工具调用，逐个执行，再发送结果。Claude Code 的流程是重叠的：模型还在生成文字的同时，已解析完成的工具调用已经在执行。对于一次包含 3 到 4 个工具调用的响应，省掉的正是逐个串行等待的那段时间。

关于错误恢复

数据流中隐藏着多层错误恢复机制。第一，API 错误，比如 429 限速和 529 服务过载：withRetry 层自动进行指数退避重试，严重情况下可以降级到备选模型。第二，上下文过长，即 prompt_too_long：触发 reactive compact，紧急执行一轮压缩，然后重试 API 调用。第三，工具执行失败：错误信息被包装为 tool_result 并标记 is_error 为 true，返回给模型，模型可以自行决定是重试还是换一种方法。query.ts 第 123 行的 yieldMissingToolResultBlocks() 确保每个 tool_use 都有对应的 tool_result，即使在中断场景下也不会出现消息配对缺失。

贯穿全程的是同一个模式：数据通过嵌套的 async generator 流动。每一层都在 generator 管道上添加自己的处理逻辑——权限检查、压缩、错误恢复——但对上层来说，它只是一个统一的事件流。关注点因此完全分离：QueryEngine 不需要知道压缩细节，REPL 不需要知道错误恢复逻辑。

1.6 启动流程

Claude Code 的启动把大量工作并行化和延迟化。整个流程分为 9 个阶段，关键路径仅约 235ms。首先，第 1 阶段是模块求值，耗时接近 0ms，处理 --version、MCP、Bridge 等快速路径；第 2 阶段是模块加载，约 135ms，MDM 读取和 Keychain 预取并行进行，同时加载 Commander、analytics 和 auth。然后，第 3 阶段是 CLI 解析，约 10ms，检测运行模式并急加载 settings；第 4 阶段是 Commander 设置，约 5ms；第 5 阶段是 preAction，约 100ms，等待 MDM、Keychain 和核心初始化完成，并做配置迁移；第 6 阶段做 14 步幂等初始化，约 100 到 200ms，涵盖配置验证、TLS、优雅关闭、OAuth 刷新、网络配置和 API 预连接，全部幂等且 memoized；第 7 阶段是 Action Handler，提取选项、验证模型并启动 REPL。最后，第 8 阶段在首帧之后延迟预取用户信息、文件计数和模型能力，不阻塞首次渲染；第 9 阶段在 Trust Dialog 之后做遥测，懒加载 400KB 以上的 OpenTelemetry。

为什么分 9 个阶段？

这个设计的核心目标是最小化用户感知的启动时间。用户关心的是输入 claude 后多快看到输入提示符，而不是所有初始化都完成了。所以有四个要点。第一，第 1、2 阶段并行预取：MDM 即移动设备管理，它的策略读取和 Keychain 凭证预取在模块加载的同时就并行启动，而不是等加载完成后串行执行。第二，第 6 阶段幂等初始化：init() 函数是 memoized 的，重复调用无副作用，这让多个代码路径都可以安全地 await init() 而不用担心重复初始化。第三，第 8 阶段延迟非关键任务：用户信息查询、文件计数统计、模型能力检测——这些对首次交互不重要的操作被推迟到首帧渲染之后。第四，第 9 阶段懒加载重依赖：OpenTelemetry，约 400KB 以上，在用户完成 Trust Dialog 之后才加载，避免拖慢启动速度。

1.7 架构总览

首先，用户终端连接 CLI 入口 main.tsx，它分出三条路径：REPL 交互模式、-p 单次查询模式，以及 SDK 或 Bridge 模式。REPL 直接进入 query 核心循环；-p 和 SDK 路径先经 QueryEngine 会话引擎，再汇入核心循环。然后，核心循环向下调用三个子系统：API 服务层，带重试降级和提示词缓存；工具系统，约 55 个工具，涵盖 BashTool、文件操作工具、AgentTool 子 Agent 和 MCP 桥接工具；上下文系统，负责系统提示词、CLAUDE.md 和 4 级压缩管道。最后，OAuth、历史持久化、遥测和插件系统作为基础设施，支撑所有其他层。

这套架构看起来像一个普通的分层架构，但每一层的设计决策都值得理解。

入口层，即 main.tsx：CLI 入口处理三种截然不同的运行模式——REPL 是交互式终端；Print 模式对应 -p 标志，单次查询后退出；SDK 和 Bridge 模式供第三方程序调用。关键设计是三种模式最终都汇聚到同一个 query() 核心循环——交互式由 REPL 直接进入，-p 和 SDK 先经 QueryEngine 会话层。这意味着核心 Agent 循环是模式无关的——无论 Claude Code 是被人类交互使用、被 CI 脚本以 -p 调用、还是被 IDE 插件通过 SDK 集成，底层执行的都是同一个 query() 函数。测试也因此省事：Print 模式就是一个现成的无头测试工具。

会话层，即 QueryEngine：管理一次对话的完整生命周期——消息在每次交互后自动持久化，累计 token 和美元开销做成本追踪，按 task budget 限额执行预算，结构化输出失败时负责重试。在无头、-p 和 SDK 路径上，它是"用户输入"和"Agent 执行"之间的边界，交互式则由 REPL 承担这一步。当一条新消息到达时，QueryEngine 判断它是斜杠命令、文件附件还是普通 prompt，做相应的预处理后才转交给核心循环。

核心循环，即 query：这是 Claude Code 的心脏。一个 while (true) 循环反复执行同一套动作：压缩上下文，调用 API，执行工具，判断是否继续。循环携带可变的 State 对象，定义在 query.ts 第 204 行，包括消息历史、压缩追踪状态、输出 token 恢复计数、turn 计数等。循环有 7 个不同的继续点，即 Continue Sites，分别处理正常工具循环、上下文过长恢复、压缩触发重试等场景。

服务层，包括 API、工具和上下文：三个独立的子系统，由核心循环编排协调。API 服务处理模型通信，包括流式传输、重试策略和提示词缓存。工具系统提供约 55 种能力，包括文件操作、搜索、Agent 派生和 MCP 桥接。上下文系统负责构建系统提示词、注入 CLAUDE.md 内容、管理 git 状态信息。三者之间互不依赖。

基础设施层，包括 OAuth、History、Telemetry 和 Plugins：横切关注点，支撑所有其他层但不参与主循环。OAuth 处理认证；History 提供对话持久化和恢复，对应 claude --resume 命令；Telemetry 追踪使用数据，采用懒加载，约 400KB 以上；Plugins 扩展工具和 Hook。

模块依赖规则

这个分层有一条关键的依赖规则：核心循环，即 query.ts，依赖服务层，但永远不依赖 UI 层——query.ts 不 import REPL.tsx，所以你可以把整个终端 UI 替换为 Web UI，核心循环及其以下的所有模块完全不用改动。反过来的依赖是单向的：交互式 UI 即 REPL.tsx，直接 import 并驱动 query()，位置在 REPL.tsx 第 146 行的 import 语句；无头、-p 和 SDK 路径先经 QueryEngine 会话层，再进入同一个 query()。SDK 模式就是后者的直接体现：它不走终端 UI，直接通过 QueryEngine 与核心循环交互。

1.8 代码规模参考

第一，TypeScript 文件约 1332 个。第二，TSX 文件，即 React 文件，约 552 个。第三，总行数在 512000 行以上。第四，内置工具约 55 个，注册表是 tools.ts，快照约 40 个 buildTool 定义。第五，Hook 事件类型有 23 种以上。第六，安全验证器 23 项。第七，MCP 传输类型 7 种。第八，权限模式 5 加 2 种。第九，内置技能 14 个。

下一章：系统主循环。
