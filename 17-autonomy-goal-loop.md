---
title: 第 17 章：自治与续跑（朗读版）
---

第 17 章：自治与续跑

本文是 docs/17-autonomy-goal-loop.md 的朗读版，表格与代码已转为口语描述，内容未增删。

第 17 章：自治与续跑——/goal 与 /loop

从这一章起，进入一个新模块：泄露快照之后的新功能。

前十六章读的是那份 3 月底流出的源码快照，但 /goal、/loop、dynamic workflow、auto mode 这些能力全在快照之后，没有源码可看。搞清楚它们只能换个办法：装上最新版真的去用，把它发出的网络请求抓下来，再对照官方文档，三头对上。所以下面凡是引号里的原文，都是这么抓出来的；凡是讲到“它内部大概怎么调度”，那是从行为反推的，讲到时会说明。这一章说清楚 /goal 和 /loop 怎么让 Claude 自己接着干，用到的办法写在文末，照着你也能自己去抓下一个功能。

17.1 两种“让它接着干”的范式

到 2026 年，Claude Code 早已不只是“一问一答”。它有一整族能力，让 agent 跨 turn、跨时间、跨会话地自己接着干。这一族里最外层、你最常碰到的两个入口，是 /goal 和 /loop——而它俩恰好是两种相反的思路。

/goal 是“盯着一个条件、不达成不罢休”。你给它一句完成条件，它就一轮一轮干下去，每轮结束由一个独立的裁判判一次“达成没有”。没达成就带着裁判给的理由再来一轮，达成了才停。它是被动的：什么时候停，交给裁判说了算。

/loop 是“定个闹钟、反复来”。你给它一个间隔，或者让它自己定节奏，它就按点反复跑同一件事。它是主动的：什么时候再来，由调度决定，跟“有没有干成”无关。

一个靠“守门人”决定何时收手，一个靠“闹钟”决定何时再来——抓住这条区别，也就抓住了 Claude Code 自治的两条主线。下面分别拆。

17.2 /goal：一个守门的裁判

它其实是 Stop hook 的语法糖

官方文档一句话点明了机制：/goal 是对一个会话级 Stop hook 的封装。每当一个 turn 结束，系统就把“你设的条件加上到目前为止的对话”发给一个评估器模型。官方称它“小快模型”、默认 Haiku，一次抓包看到的实际模型见下。它回一个“是或否，外加一句理由”。答“否”，就让 Claude 带着这句理由再干一轮；答“是”，就清掉目标、并在会话记录里记一笔达成。

抓包印证了这套流程。你设目标那一刻，主模型收到的消息里有这么一段，几乎是把机制写在了明面上。下引为关键片段，末尾省略：

A session-scoped Stop hook is now active with condition: “你的条件”. Briefly acknowledge the goal, then immediately start (or continue) working toward it — treat the condition itself as your directive and do not pause to ask the user what to do. The hook will block stopping until the condition holds. It auto-clears once the condition is met…

“设目标即启动一轮，把条件本身当指令。”这就是为什么你不用再单独发一句提示。

裁判的判决：三种结果，一道死循环刹车

真正有意思的是那个裁判。把它发出的真实请求抓下来，它的系统提示词逐字就是下面这段。不长，值得整段读一遍，因为一个自治循环的全部分寸都压在这几行里：

You are evaluating a stop-condition hook in Claude Code. Read the conversation transcript carefully, then judge whether the user-provided condition is satisfied.

Your response must be a JSON object with one of these shapes. 第一种形状：ok 为 true，reason 里引用记录中满足条件的证据。第二种形状：ok 为 false，reason 里写明还缺什么、或是什么挡住了条件。第三种形状：ok 为 false、impossible 为 true，reason 解释条件为什么永远无法满足。

Always include a “reason” field, quoting specific text from the transcript whenever possible. 如果记录里没有清楚证据表明条件已满足，就返回 ok 为 false，reason 写 insufficient evidence in transcript。

Only use impossible when the condition is genuinely unachievable in this session — for example: the condition is self-contradictory, it depends on a resource or capability that is unavailable, or the assistant has explicitly tried, exhausted reasonable approaches, and stated it cannot be done. Apply your own judgment when deciding this — the assistant claiming the goal is impossible is evidence, not proof; independently confirm the condition is genuinely unachievable rather than deferring to the assistant's self-assessment. Do not use it just because the goal has not been reached yet or because progress is slow. When in doubt, return ok 为 false，不要带 impossible。

三种结果——达成、没达成、判定不可能——就是这段里的三个 JSON 形状。前两种直白；关键在第三种。impossible 是一道精心设计的死循环刹车，而整整一段都在提防同一件事：别让主 agent 把裁判忽悠着提前认输。“主 agent 说干不成，只算证据、不算铁证；裁判得自己独立确认，拿不准就返回 ok 为 false、别加 impossible。”一个自治循环最怕的就两头——要么停不下来，要么被内部说服着草草收场，这段提示词正是同时冲着这两头写的。它甚至连“进度慢”都点名排除：慢不等于不可能。

三个抓包才看得到的工程细节

把裁判的真实请求整个拆开，还能看到官方文档没提的三件事。

第一，判决是 API 层强制的，不只是提示词请求。这条请求带了一个 output_config，用 JSON schema 把输出死死约束成 ok、reason、impossible 三个字段的形状，其中 ok 和 reason 必填、不许有别的字段。提示词只是说明，schema 才是护栏——就算模型想自由发挥，也发挥不出这个形状之外。

第二，裁判不给工具、只让它看对话。请求里的 tools 是空的。这印证了官方那句“裁判不调用工具，只能判断已经出现在对话里的内容”：它虽然被塞了一个 transcript 路径，却没有任何工具去读那个文件。

第三，裁判跑在高推理档。请求里 effort 字段是 high。判“到底达没达成”这件事，系统舍得花算力。

关于用哪个模型，有一处得说老实话：官方文档说裁判用“你配置的小快模型，默认 Haiku”，但我这台机器抓到的实际是 claude-fable-5，因为本机把小快模型配成了它。所以别把一次抓包看到的模型名当成默认值。还要澄清一点：是客户端的 hook 运行时在本地组装并发出这条请求，模型推理仍在 Anthropic 那边跑，不是你本地在跑模型。

它在追踪里长什么样

/goal 每一轮的进度都会落进会话记录：一条 goal_status，带着条件、迭代了几轮、耗时、烧了多少 token、达成没。所以“一个目标跨多少 turn、每轮什么状态”是完全可回放的。这正是上一章讲的那份“默认就在”的追踪的一个活例子，详见第 16 章。

17.3 /loop：一个自己排程的闹钟

/loop 和 /goal 骨子里不一样。它不是一个被动 hook，而是一大段由主模型执行的编排提示词。你敲 /loop 加你的输入，系统就注入一段编排指令，开头一行是 /loop — schedule a recurring or self-paced prompt。它让主模型自己去解析、选调度方式、调用通用的调度工具。换句话说，/loop 的“聪明”写在提示词里，不是一个硬编码的调度器。不过要补一句，真正的执行、生命周期和保护栏仍然硬编码在运行时，这点后面会看到。

它怎么解析你输入的

抓到的这段编排提示词，把解析规则逐字写死了。直接看原文，比我转述清楚：

原文开头一行是标题：/loop — schedule a recurring or self-paced prompt。接着的解析段标着按优先级排序，列了三条规则。第一条叫 Leading token，看开头的词：如果输入第一个以空白分隔的词，是数字加时间单位的形式，比如 5m、2h，那它就是间隔，其余部分是提示词。第二条叫 Trailing every clause，看结尾的 every 从句：否则，如果输入以 every 加数字加单位、或 every 加数字加单位词结尾，比如 every 20m、every 5 minutes，就把这段抽出来当间隔，并从提示词里去掉。只有当 every 后面跟的是时间表达式才匹配——check every PR 就没有间隔。第三条叫 No interval，没有间隔：否则，整个输入就是提示词，你会动态自定节奏。

示例部分举了三个例子。check the deploy every 20m，解析出间隔 20m、提示词 check the deploy，走第二条规则。check every PR，没有间隔，进动态模式，提示词就是 check every PR，走第三条规则，因为 every 后面跟的不是时间。只敲一个 5m，提示词为空，显示用法。

三条规则一目了然：先看开头是不是 5m 这种间隔，再看结尾有没有 every 20m 从句，都没有就整句当 prompt、进自定节奏。最见功力的是 check every PR 那个反例——提示词特意声明“every 后面不是时间表达式就不算间隔”，免得把“检查每个 PR”误读成“每隔一个 PR 跑一次”。把边界情形连同用例一起写进提示词，这本身就是一课。

三条执行路径

解析出间隔和 prompt 后，走哪条路也是提示词说了算。

如果间隔在一小时以上，或者你用的是“每天早上”“每晚”这种日级措辞，提示词让模型先弹一个问题，问你要不要改成能脱离会话长期运行的云端计划，也就是 /schedule。因为纯 /loop 一关会话就没了。你若选“只在本会话”，它才继续往下。

要是就在本会话按固定间隔跑，模型会调 CronCreate。这一步我抓到了真实调用：参数里 cron 字段是每分钟触发一次的标准表达式，prompt 字段是要反复执行的那句话，recurring 字段设为 true。

工具返回的关键部分是：Scheduled recurring job 7e7d1261 (Every minute). Session-only (not written to disk, dies when Claude exits). Auto-expires after 7 days. Use CronDelete to cancel sooner. 一句话里印证了好几点。1m 被转成了标准 cron 表达式，就是每分钟触发一次。任务有个 8 位 id，只活在会话里、不落盘，所以 Claude 一退出就没了、也不留垃圾。还会在 7 天后自动过期。

有个容易误解的点值得单独点出：建完 cron 之后，编排提示词要求模型立刻先把解析出的任务跑一次。原文是 “Then immediately execute the parsed prompt now — don't wait for the first cron fire.” 所以 /loop 1m X 是先安排好定时、再马上执行一次 X，不是等一分钟后才第一次跑。之后才由 cron 继续按点触发。

如果你没给间隔，走的是“自定节奏”：模型用一个叫 ScheduleWakeup 的工具自己排下一次醒来。这个工具的描述里把设计取舍讲得很透，值得引一段原文：

Do NOT schedule a short-interval wakeup to poll for background work you started — when harness-tracked work finishes, you are re-invoked automatically, so polling is wasted. Instead schedule a long fallback (1200s+) so the loop survives if the work hangs or never notifies.

翻成人话：别拿短间隔去反复查那些 harness 本来就会在完成时通知你的后台活——那纯属白轮询。应该排一个较长（一千多秒起）的兜底唤醒，只为防止任务挂死或永远不回信。每次还要把同一个 /loop prompt 通过参数传回去，下一次醒来才接着干同一件事。一个让模型自己给自己定闹钟的工具，居然要先教模型“别把闹钟定太勤”，这恰恰暴露了自排程最容易踩的坑。

四种输入、四条路径，摆到一起更清楚。

第一，你输入 1m X 这种带间隔的形式，走的是建会话内定时任务这条路。第一步先立刻把 X 跑一次，之后由定时任务按点重新投递 X。

第二，你输入不带间隔的 X，走自定节奏。第一步同样是先立刻跑一次 X，之后用 ScheduleWakeup 或 Monitor 安排下一次。

第三，什么都不输入，走内置维护或 loop.md。先跑默认的维护 prompt，之后同样按节奏继续。

第四，间隔在一小时以上、或者用了“每天”这类措辞，先弹问题、提议改成云端计划。你选了只在本会话，才继续上面几条路。

什么都不给它，它跑维护脚本

/loop 后面要是空着，据官方文档它会跑一个内置的维护提示词，或者你自己写的 loop.md。优先接着做没做完的事，照看当前分支的 PR，闲下来做点清理；并守一条底线——不开新战线，推送、删除这类不可逆动作只在对话里已授权时才做。这段是照官方文档讲的。二进制里能搜到相邻的 babysit、autofix 片段，但不足以逐字坐实“修失败 CI、回 review、解合并冲突”就是空 /loop 的通用默认。所以按官方描述转述，别把它当成从产物里直接确认过的行为。

背后的调度守护进程

本地的 /loop、cron、动态唤醒和通知路径，背后能看到一组同族的后台守护进程遥测，内部叫 KAIROS。至于云端计划是不是也由这个客户端 daemon 管，证据不足。而且官方口径是云端跑在 Anthropic 那边，所以这里只把 KAIROS 当作本地调度、唤醒底座的一部分。它的内部调度和持久化逻辑是纯客户端后台代码，抓不到、只能推断。

准确的说法是这样：/loop 的高层决策，也就是怎么解析、选哪条路、什么时候收敛，由提示词主导。但执行、生命周期和保护栏在运行时里写死。比如 ScheduleWakeup 的延迟被夹在一个固定区间内，加上保活兜底、太久没动就回收、cron 存储、调度器本身，这些都不是模型能改的。提示词负责“决策”，运行时负责“兜底”。

最后一处值得记一笔，因为它是“文档和实测对不上”的一个例子。关于 jitter，也就是把触发时间随机错开、防止一堆会话同一秒打 API，官方文档说重复任务最多可以滞后半小时。但我这台 v2.1.201 机器上，CronCreate 工具自己的描述写的是“滞后周期的 10%、最多 15 分钟”。两个口径对不上，多半是版本差异——引用时得注明是哪个来源。

17.4 两个入口，一族底座

/goal 和 /loop 只是最外层的两个入口，它俩摆在一起，正好照出这一族自治能力的两个维度：谁触发下一轮，以及靠什么调度。下面点到的其它成员是同一族里相邻的能力，并非每一个都在本章里被抓包验证过。这里只把它们跟 /goal、/loop 的关系串一下，完整分析留给后面的章节。

触发这一侧，除了 /goal 封装的 Stop hook，还有 auto mode。它逐个自动放行工具调用，把每一步的确认省掉，好让续跑的每一轮都能无人值守地跑完。调度这一侧，除了 /loop 用到的 cron 和 ScheduleWakeup，还有 Monitor，以及把这些串起来的 KAIROS 守护进程。Monitor 盯着一个长跑脚本、有输出就推回来，比轮询更省。再往外，还有云端计划，它跑在 Anthropic 那边、关了会话也不停。还有后台 agent，也就是 /bg 和 claude agents。以及 --resume 时把没做完的目标和没过期的定时任务一起复活。

一句话概括这一族：一个后台调度基座，加一套通用调度工具，加提示词编排。/goal 和 /loop 是这套底座露在外面的两个语法糖，后面的章节会接着拆这一族的其它成员。

17.5 附录：我们怎么知道的（逆向方法，可复现）

本章能确认的东西，都来自两层办法。下面把过程写全，任何人都能复现——你甚至可以把这一节直接丢给 Claude Code，让它照着帮你重做一遍。

第一层是静态抽取，零成本、发版当天就能做。要提醒一句：2.1.x 起，Claude Code 已经不是一个明文的 cli.js，而是编译成了原生二进制，用 Bun 打包。但 JS 源仍作为字符串嵌在里面，用 strings 工具照样抠得出来。抠出全部字符串后，用 grep 找某个功能的遥测事件名，当功能地图用，再找它的提示词和命令本体。有一个坑：在这个几十万行的大文件上，点加区间量词那种正则会疯狂回溯、直接超时，要用定长字符串匹配的 grep -F。这一层能拿到提示词片段、工具描述、事件名，都是逐字能核对的。但拿不到这些字符串怎么拼装、控制流长什么样——那些只能靠推断。

第二层是抓真实网络请求，看运行时最终组装出来的完整报文，系统提示、消息、工具、模型、output_config 一应俱全。做法是架一个明文反向代理：让 claude 明文连本地，这样免去证书那套麻烦；代理把请求体记下来，再转发到真的 API。跑的时候从一个“已信任”的目录起，/goal 这类 hook 系功能要求工作目录接受过信任对话。把 ANTHROPIC_BASE_URL 指到本地代理，然后执行 claude -p “/goal …”。在抓到的一堆请求里，找带那句 stopping condition 的，就是裁判那条。先拿一句 -p “reply with PONG” 冒烟验证代理通了、能抓到，再跑目标功能。探针条件挑那种一句话就能满足、很快收敛的，别动工具权限、别长跑烧 token。

还有两层顺手就能补。一是会话记录：跑完直接读 ~/.claude/projects/ 下的 JSONL，里面有 goal_status、每一步的 token 用量、消息间的父子链。这是追踪那一层的证据，零成本就能拿到。二是 OTel 遥测：开 CLAUDE_CODE_ENABLE_TELEMETRY，加 console 导出器跑一遍，能看到那些内部 tengu_ 开头的事件，对外导成 claude_code 开头的指标，真的在发。

最后是纪律。提示词、报文、字符串是能逐字核对的硬证据；内部怎么调度、怎么放行，是从行为反推出来的，只能当推断，不能拿它当确凿的事实来引。模型名、耗时这种只观测到一次的值，要写明是“这一次抓到的”，别当成通例。事件名要逐个列全，别用 a 竖线 b 竖线 c 这种正则式简写——一压缩就可能拼出一个根本不存在的名字。还有一条安全提醒：抓下来的报文里有账号、设备、会话的 id，还有提示词和工具结果，可能夹着密钥。只自己私下留着，要引用先把敏感信息抹掉。

想复现的话，一句话就能交给 Claude Code：“读这一节的方法，帮我复现 /goal 裁判的逆向——用 strings 抠本机 claude 二进制里 goal 相关的字符串；写一个明文反代，让 claude -p ‘/goal …’ 走它、抓到裁判那条请求；把裁判的系统提示、output_config、tools、模型抠出来，跟官方 /goal 文档对一遍，标清哪些是抓到的原文、哪些是推断出来的。”这就是全部办法，你现在也能自己去抓下一个新功能了。

附录：完整提示词原文（逐字）

正文只摘了关键处；下面把从抓包里逐字取出的完整原文放全，供想深挖或复现的人对照。来源是对 2.1.201 客户端的明文反代抓包，凡属注入槽位，也就是你的条件、你的输入，已替换成占位符，其余一字未改。

/goal 评估器系统提示词（stop-condition evaluator system prompt）

（代码从略：这段是 /goal 评估器系统提示词的逐字原文，与 17.2 节引用的判决提示词一致。要点是让模型细读对话记录后判断条件是否满足，回复限定为 ok 加 reason 的三种 JSON 形状之一，第三种加 impossible 表示条件在本会话内无法达成。提示词反复叮嘱，助手自称做不到只是证据不是证明，要独立确认，拿不准就只返回 ok 为 false、不加 impossible。）

/goal Stop hook 激活消息（设目标时注入主模型，your condition 是你填的条件）

（代码从略：这段是设目标那一刻注入主模型的激活消息逐字原文。要点是告知会话级停止条件已激活，让模型简短确认目标后立即朝它开工，把条件本身当指令、不要停下来问用户该做什么。hook 会阻止停止、直到条件成立；条件达成后目标自动清除，成功后不必让用户去跑 /goal clear，那条命令只用于提前清除目标。）

/loop 命令编排提示词（Input 段是把你的输入拼进去的位置）

（代码从略：这段是 /loop 命令的完整编排提示词。要点：先按优先级解析输入，开头的时间 token 算间隔，结尾的 every 时间从句算间隔，都没有就整句当提示词、进动态模式，并附了正反示例。解析出 60 分钟以上的间隔、或“每天早上”这类日级措辞时，先用选择题问用户要不要改用云端计划，选云端就直接调 schedule 技能并停止，选只在本会话才继续。固定间隔模式把间隔换算成 cron 表达式，其中有一张间隔与表达式的对照小表，除不尽就就近取整并告知用户，然后调用 CronCreate、报告任务信息，并立刻先把任务执行一次、不等第一次定时触发。动态模式先跑任务，事件驱动就布一个 Monitor，本轮最后一个动作是调 ScheduleWakeup，把原始 /loop 输入原样传回去，靠通知醒来就先处理事件、再排下一次。要停止就不调 ScheduleWakeup、停掉 Monitor，停止前先发一条推送告知结果。最后是 Input 段，你的输入就拼在这里。）

ScheduleWakeup 工具描述（dynamic 自定节奏模式的核心）

（代码从略：这是 ScheduleWakeup 工具的描述原文，动态自定节奏模式靠它排下一次醒来。要点：不要用短间隔轮询 harness 完成时会自动通知的后台任务，改排 1200 秒以上的长兜底；harness 追踪不到的外部任务例外，按状态实际变化的速度挑延迟。每轮都要把同一个 /loop 提示词原样传回，自主循环则传一个专用占位符；不调用这个工具就结束循环。delaySeconds 的挑选围绕提示词缓存的 5 分钟窗口来讲：5 分钟以内缓存还热，适合轮询外部状态；5 分钟到 1 小时付一次缓存未命中，适合没理由更早查的场景。描述特别警告别选正好 300 秒，两头不讨好；没有信号的空闲心跳默认 1200 到 1800 秒。运行时会把延迟夹在 60 到 3600 秒之间。reason 字段用一句话写明为什么选这个延迟，会进遥测并展示给用户。）

CronCreate 工具描述（固定间隔、定时任务）

（代码从略：这是 CronCreate 工具的描述原文，管固定间隔和一次性定时任务。要点：用用户本地时区的标准 5 字段 cron 表达式。一次性任务触发一次就自动删除；重复任务默认开启，7 天后自动过期，到期前可用返回的任务 id 提前取消。任务只活在当前会话、不落盘，会话退出就没了。要盯日志、进程或命令输出的实时变化，应该改用 Monitor 工具，cron 只是按表轮询。任务只在 REPL 空闲时触发；调度器会加一点确定性的抖动，重复任务最多滞后周期的 10%、上限 15 分钟，一次性任务落在整点或半点、最多提前 90 秒触发。描述还特别提醒，任务允许时避开整点和半点的分钟数，免得全球用户的请求在同一瞬间打到 API。）
