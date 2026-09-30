---
title: 速查参考（朗读版）
---

速查参考

本文是 docs/reference.md 的朗读版，表格与代码已转为口语描述，内容未增删。

一页搞定：核心概念、常用工具、关键源码入口。

核心概念速查

第一个概念是 Agent Loop，一句话解释：它是从用户输入到模型决策，再到工具执行和结果注入的循环，直到模型返回纯文本。详见第 2 章。

第二个概念是 query 函数，一句话解释：它是核心循环的异步生成器实现，包含 7 个 continue site，处理不同的恢复策略。详见 2.4 节。

第三个概念是 QueryEngine，一句话解释：它是会话级管理器，驱动 query 函数并处理预算、权限和结构化输出。详见 2.3 节。

第四个概念是 Autocompact，一句话解释：它是 Token 使用量接近上下文窗口时的自动压缩机制。有效窗口利用率约 93%，大约在原始窗口的 83% 时触发。详见 3.6 节。

第五个概念是 Context Collapse，一句话解释：它是投影式只读上下文折叠，不修改原始消息，可安全回退。详见 3.7 节。

第六个概念是 CLAUDE.md，一句话解释：它是项目级指令文件，从 CWD 向上遍历目录树发现，支持多层级。详见 3.2 节。

第七个概念是 buildTool 函数，一句话解释：它是工具工厂函数，合并 TOOL_DEFAULTS 和工具定义，其中 TOOL_DEFAULTS 提供 fail-closed 默认值。详见 4.1 节。

第八个概念是 MCP，全称 Model Context Protocol，一句话解释：它是外部工具扩展协议，支持 6 种传输机制，服务端配置类型有 8 种。详见 4.9 节。

第九个概念是 ToolSearch，一句话解释：它是延迟加载机制，50 多个工具中只按需加载，减少每次 API 调用的 prompt 体积。详见 4.10 节。

第十个概念是 search-and-replace，一句话解释：它是 FileEditTool 的编辑策略，要求 old_string 在文件中唯一匹配。详见第 10 章。

第十一个概念是纵深防御，一句话解释：它是 7 层独立安全检查，任一层被绕过不致命。详见第 12 章。

第十二个概念是 Plan 模式，一句话解释：它是两阶段执行，先只读探索，再经用户审批，最后可写实施。详见 8.6 节。

第十三个概念是协调器模式，一句话解释：主 Agent 只编排不执行，通过 Worker 完成实际任务。详见 8.3 节。

第十四个概念是 Hooks，一句话解释：它是事件驱动扩展机制，在工具执行生命周期的关键节点注入自定义逻辑。详见第 7 章。

常用工具清单

文件操作

第一个工具是 Read，即 FileReadTool，只读，并发安全，用来读取文件，支持行范围、PDF 和图片。

第二个工具是 Write，即 FileWriteTool，不是只读，并发不安全，用于写入或创建文件。

第三个工具是 Edit，即 FileEditTool，不是只读，并发不安全，采用 search-and-replace 编辑，要求唯一匹配。

第四个工具是 NotebookEdit，不是只读，并发不安全，用于编辑 Jupyter Notebook。

搜索与导航

第一个工具是 Glob，即 GlobTool，只读，并发安全，按模式匹配搜索文件名。

第二个工具是 Grep，即 GrepTool，只读，并发安全，对文件内容做正则搜索，基于 ripgrep。

第三个工具是 ToolSearch，即 ToolSearchTool，只读，并发安全，动态发现延迟加载的工具。

执行与系统

第一个工具是 Bash，即 BashTool，不是只读，并发不安全，执行 Shell 命令，配合 tree-sitter AST 和 23 项静态安全检查。

第二个工具是 Agent，即 AgentTool，不是只读，并发不安全，派生子 Agent 执行独立任务。

第三个工具是 SendMessage，不是只读，并发不安全，向已有 Agent 或队友发送消息。

第四个工具是 TaskStop，不是只读，并发不安全，用于终止子 Agent。

模式控制

第一个工具是 EnterPlanMode，进入 Plan 模式，即只读探索阶段。

第二个工具是 ExitPlanMode，退出 Plan 模式，并提交计划供审批。

关键源码入口

第一个入口是 CLI 入口，入口文件 src/main.tsx，约 4700 行，职责是 Commander.js 参数解析和运行模式分发。

第二个入口是 Agent 循环，入口文件 src/query.ts，约 1730 行，职责是核心循环的异步生成器实现。

第三个入口是会话管理，入口文件 src/QueryEngine.ts，约 1300 行，职责是对话生命周期管理。

第四个入口是工具接口，入口文件 src/Tool.ts，约 790 行，职责是 Tool 类型定义和 buildTool 工厂。

第五个入口是系统提示词，入口文件 src/constants/prompts.ts，约 910 行，职责是系统提示词模板。

第六个入口是权限系统，入口文件是 src/utils/permissions 目录，含多个文件，职责是多层权限检查和规则匹配。

第七个入口是 Bash 安全，入口文件 src/tools/BashTool/bashSecurity.ts，约 2600 行，职责是 23 项静态安全验证器。

第八个入口是上下文组装，入口文件 src/context.ts，约 190 行，职责是系统和用户上下文构建。

第九个入口是压缩服务，入口文件是 src/services/compact 目录，含多个文件，职责是 Autocompact、Snip、Context Collapse。

第十个入口是 MCP 客户端，入口文件 src/services/mcp/client.ts，约 3350 行，职责是 MCP 连接管理和工具注册。

第十一个入口是 Hooks 引擎，入口文件是 src/hooks 目录，含多个文件，职责是 Hook 事件分发和执行。

第十二个入口是多 Agent，入口文件 src/coordinator/coordinatorMode.ts，约 370 行，职责是协调器模式实现。

第十三个入口是 Swarm 后端，入口文件是 src/utils/swarm/backends 目录，含多个文件，职责是 Tmux、iTerm2、InProcess 执行后端。

关键阈值与常量

第一个常量是 AUTOCOMPACT_BUFFER_TOKENS，值为 13000，来源 autoCompact.ts，用途是自动压缩触发缓冲。

第二个常量是 MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES，值为 3，来源 autoCompact.ts，用途是压缩熔断器阈值。

第三个常量是 CAPPED_DEFAULT_MAX_TOKENS，值为 8000，来源 context.ts，用途是默认输出 token 上限，节省 slot。

第四个常量是 ESCALATED_MAX_TOKENS，值为 64000，来源 context.ts，用途是截断后升级的输出上限。

第五个常量是 MAX_OUTPUT_TOKENS_FOR_SUMMARY，值为 20000，来源 autoCompact.ts，用途是为压缩摘要预留输出空间。

第六个常量是 DEFAULT_MAX_RESULT_SIZE_CHARS，值为 50000，来源 toolLimits.ts，用途是工具结果最大字符数。

第七个常量是 MAX_TOOL_RESULT_TOKENS，值为 100000，来源 toolLimits.ts，用途是工具结果最大 token 数。

第八个常量是 DENIAL_LIMITS.maxConsecutive，值为 3，来源 denialTracking.ts，用途是连续拒绝后回退到交互确认。

第九个常量是 DENIAL_LIMITS.maxTotal，值为 20，来源 denialTracking.ts，用途是总拒绝数上限。

第十个常量是 WARNING_THRESHOLD，值为 0.7，即 70%，来源 rateLimitMessages.ts，用途是速率限制警告阈值。

第十一个常量是 POST_MAX_RETRIES，值为 10，来源 SSETransport.ts，用途是 POST 请求最大重试次数。

第十二个常量是 RECONNECT_GIVE_UP_MS，值为 600000 毫秒，即 10 分钟，来源 SSETransport.ts，用途是 SSE 重连放弃时间。

第十三个常量是 LIVENESS_TIMEOUT_MS，值为 45000 毫秒，来源 SSETransport.ts，用途是心跳超时，服务端每 15 秒发送心跳。

返回：快速入门和首页。
