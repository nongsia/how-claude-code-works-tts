---
title: 第 10 章：代码编辑策略（朗读版）
---

第 10 章：代码编辑策略

本文是 docs/05-code-editing-strategy.md 的朗读版，表格与代码已转为口语描述，内容未增删。

会写代码只是起点，好用的 coding agent 还得会用低破坏性的方式改代码。

代码编辑是 coding agent 最关键也最危险的能力——一次错误的编辑可能破坏整个代码库，而一套好的编辑策略能让 agent 像经验丰富的开发者一样精准地改代码。Claude Code 的编辑策略围绕三个原则设计：最小化破坏性，只改需要改的部分；可验证性，每次编辑都有明确的改动前与改动后；抗幻觉，模型无法静默写入不存在的代码。search-and-replace 式的 FileEditTool 天然满足这三个约束，所以 Claude Code 优先使用它；全文件覆盖的 FileWriteTool 只留给创建新文件或需要完整重写的场景。

10.1 两种编辑工具

Claude Code 提供两种文件编辑工具，各有其适用场景。第一种是 FileEditTool，采用 search-and-replace 策略，适合修改已有文件中的特定部分，破坏性低。第二种是 FileWriteTool，采用全文件覆盖写入策略，适合创建新文件或完整重写，破坏性高。

系统提示词与工具描述共同引导模型优先使用 FileEditTool——工具描述里更是写死了"只在创建新文件或完整重写时才用 Write"。

FileEditTool 之所以成为默认选择，是因为它在工程上解决了 LLM 编辑代码最棘手的问题：如何让一个可能产生幻觉的模型安全地修改真实代码？下面逐层拆开 FileEditTool 的设计，交代每个工程决策背后的理由。

10.2 FileEditTool：Search-and-Replace 方法

FileEditTool 是 Claude Code 代码编辑的核心工具，采用精确字符串替换策略。本节按照从外到内的顺序展开：10.2.1 先看接口设计，10.2.2 理解方案选型的工程考量，10.2.3 到 10.2.5 依次深入输入预处理、验证管线和实现细节。

10.2.1 接口设计与工作原理

输入 Schema

（代码从略：这段代码定义了编辑工具的输入格式，包含四个字段——要编辑的文件绝对路径，要替换的精确字符串 old_string，替换后的新字符串 new_string，以及一个可选的 replace_all 开关，表示是否替换所有出现位置，默认为 false。）

工作原理

FileEditTool 不需要行号、不需要正则表达式。它的工作方式极其简单。第一步，在文件中精确查找 old_string。第二步，确保 old_string 在文件中唯一出现，除非 replace_all 为 true。第三步，将其替换为 new_string。第四步，如果 old_string 不唯一，返回错误，要求提供更多上下文。

10.2.2 为什么 Search-and-Replace 优于其他方案

这个设计选择背后有几层工程考量。

第一，低破坏性。Search-and-replace 只修改目标文本，文件的其余部分完全不变。相比之下，全文件写入可能出三类问题：第一，意外丢失未预期的内容；第二，引入格式变化，比如缩进和空行；第三，在大文件上因 Token 限制截断内容。

第二，可验证性。每次编辑都有明确的改动前和改动后。用户可以精确看到什么被改了——这比看一个完整的新文件要容易得多。

第三，抗幻觉。模型需要提供文件中实际存在的精确字符串。如果模型"幻觉"了不存在的代码，编辑会直接失败并返回错误，而不是静默地写入错误内容。

第四，Token 效率。对大文件的小修改，search-and-replace 只需要发送修改点附近的上下文，而不是整个文件内容。

第十，Git 友好。Search-and-replace 产生的 diff 最小化、最精确。自动化 PR 创建时，reviewer 看到的是干净的、有针对性的变更。

与备选方案的对比

在确定 search-and-replace 方案之前，有必要理解为什么其他看似合理的方案被排除了。

基于行号的编辑，比如直接指定编辑第 42 到 45 行：这是最直觉的方案，但也是最脆弱的。问题在于行号是位置相关的——当模型在一个对话 turn 中需要对同一文件做多处修改时，第一个编辑（比如在第 10 行插入 3 行代码）会导致后续所有行号偏移。模型要么需要一个复杂的行号重算逻辑，要么只能保证每次只编辑一处。而 search-and-replace 是位置无关的：不管文件上方插入了多少行，目标字符串的内容不会变，匹配始终有效。

基于 AST 的编辑，比如把函数 foo 重命名为 bar：这个方案在理论上很优雅，但实际不可行。Claude Code 需要支持几十种编程语言，为每种语言维护一个完整的 AST 解析器成本极高。更关键的问题是：语法错误的文件恰恰是最需要编辑的文件，但 AST 解析器在遇到语法错误时会直接报错拒绝解析。这意味着在最需要编辑工具的场景下（修 bug、修语法错误），工具反而不可用。

Unified diff 格式，也就是 patch 格式，比如让模型直接输出以 @@ 开头、带行号区间的标准 diff：LLM 在生成这种严格格式时表现很差。Unified diff 要求精确的 hunk header，即起始行号和行数；要求每一行都正确使用加号、减号或空格前缀；上下文行数必须与 header 声明一致。任何一个字符的偏差都会导致整个 patch 无法应用。相比之下，search-and-replace 只需要模型提供两段自然语言级别的字符串——这正是 LLM 最擅长的任务形式。

全文件重写，也就是 FileWriteTool 的方式：对于小文件可行，但对于大文件问题严重。一个 500 行的文件，哪怕只改一行，模型也需要输出完整的 500 行内容。这不仅浪费 Token，更危险的是，模型可能在输出过程中遗漏未修改的代码——文件中间重复性强的段落，比如一连串相似的 case 语句，最容易被吞掉。而且用户无法快速 review 变更：面对一个 500 行的新文件，找到实际修改的那一行如同大海捞针。

幻觉安全是 search-and-replace 最被低估的优势。设想这样一个场景：模型"记得"文件中有一个 handleError() 函数，但实际上这个函数在上一次重构中已经被重命名为 processError()。如果使用 search-and-replace，模型提供的 old_string 写着 function handleError()，会直接失败——错误码 8，错误信息是"要替换的字符串在文件中找不到"；模型看到错误后会重新读取文件，发现正确的函数名。如果使用全文件重写，模型可能会写出包含 handleError() 的完整文件，覆盖掉正确的 processError()——而且这个错误完全是静默的，不会有任何报错。

10.2.3 输入预处理管线

在进入核心的验证和执行流程之前，模型的输入会先经过一个预处理阶段。normalizeFileEditInput() 在 validateInput 之前被调用，负责清洗模型输出中常见的瑕疵。

尾部空白裁剪

模型生成代码时经常在行尾添加多余的空格或 tab。stripTrailingWhitespace() 会对 new_string 的每一行去除尾部空白字符。但这个规则有一个重要的例外：.md 和 .mdx 文件不做尾部空白裁剪。这是因为在 Markdown 语法中，行尾的两个空格表示硬换行，也就是 br 标签的效果，裁剪掉会改变文档的语义。

（代码从略：这段代码先判断目标文件是否是 md 或 mdx 文件，是就原样保留 new_string，不是才做尾部空白裁剪。）

API 反消毒（Desanitization）

Claude API 出于安全考虑，会将某些 XML 标签"消毒"为短形式，英文叫 sanitize，防止模型输出被误解析为 API 控制标签。对应关系是：第一，模型看到的 fnr 短标签，对应文件中的 function_results 完整标签。第二，模型看到的 n 开标签和闭标签，对应 name 标签。第三，模型看到的 s 开标签和闭标签，对应 system 标签。第四，模型看到的两个换行加 H:，对应两个换行加 Human:。第五，模型看到的两个换行加 A:，对应两个换行加 Assistant:。

当模型输出的 old_string 无法精确匹配文件内容时，desanitizeMatchString() 会尝试将这些消毒后的短形式还原为原始标签。如果还原后能匹配成功，同样的替换也会应用到 new_string，确保编辑的一致性。

这个预处理阶段对用户完全透明——大多数情况下用户不会意识到它的存在。但对于编辑包含 XML 标签或 Human:、Assistant: 这类特殊字符串的文件（例如 prompt 模板文件），它是编辑能否成功的关键。

10.2.4 完整验证管线

FileEditTool 的 validateInput() 方法实现了一个多层验证管线，在真正执行编辑之前拦截各种问题。验证的顺序是刻意设计的：低成本的检查在前，需要文件 I/O 的检查在中，依赖文件内容的检查在后。这样在早期阶段就能拦截的问题，不会浪费后续的磁盘读取开销。

完整的验证步骤和对应的错误码如下。第一步，错误码 0，做团队记忆密钥检查，防止将密钥写入团队记忆文件。第二步，错误码 1，检查 old_string 是否等于 new_string，拒绝无意义的空操作。第三步，错误码 2，做权限 deny 规则匹配，尊重用户配置的路径排除规则。第四步，做 UNC 路径检测，出于安全考虑防止 Windows NTLM 凭据泄露，这一步没有专门错误码。第五步，错误码 10，检查文件大小是否超过 1 GiB，防止 V8 字符串长度限制导致内存溢出。第六步，做文件编码检测，通过 BOM 判断是 UTF-16LE 还是 UTF-8，这一步没有专门错误码。第七步，错误码 4，检查文件不存在且 old_string 非空的情况，找不到目标文件时尝试给出相似文件建议。第八步，错误码 3，检查 old_string 为空但文件已有内容的情况，阻止用"创建新文件"的方式覆盖已有文件。第九步，错误码 5，检测 ipynb 扩展名，重定向到 NotebookEditTool。第十步，错误码 6，检查 readFileState 缓存缺失或只是部分视图，文件未被读取就必须先读。第十一步，错误码 7，检查文件修改时间晚于读取时间，说明文件被外部修改，需要重新读取。第十二步，错误码 8，检查 findActualString() 返回空，也就是 old_string 在文件中不存在。第十三步，错误码 9，检查匹配数超过 1 且未开启 replace_all，也就是有多个匹配但未指定全局替换。第十四步，错误码 10，执行配置文件编辑的专用校验，对 Claude 配置文件做 JSON Schema 校验。

几个值得展开讨论的步骤：

步骤 7 和 8 构成文件创建的双重门控。old_string 为空有特殊语义——它表示"创建新文件"。当 old_string 为空且文件不存在时，验证直接通过；当 old_string 为空但文件已存在且有内容时，也就是错误码 3，会阻止操作，防止模型误用创建语义覆盖已有文件。但如果文件存在且内容为空，即文件内容去除首尾空白后是空字符串，则允许通过——这处理了"空文件等同于不存在"的边界情况。

步骤 12 是字符串查找。这一步调用前面介绍的 findActualString()，先尝试精确匹配，再尝试引号标准化后匹配。如果两种方式都找不到，返回错误码 8 并附上 old_string 的内容，帮助模型理解匹配失败的原因。同时，错误返回中还会附带 isFilePathAbsolute 元信息——因为一个常见的失败原因是模型使用了相对路径，导致在错误的目录下查找文件。

步骤 14 保护配置文件。对 .claude/settings.json 等配置文件，验证不仅检查 old_string 是否存在，还会模拟执行编辑并验证结果是否符合 JSON Schema。这防止了一个危险场景：一次看似合理的编辑可能导致配置文件格式损坏，使 Claude Code 无法正常启动。

10.2.5 唯一性约束

在上面的验证管线中，步骤 13 的唯一性约束值得单独讨论。FileEditTool 要求 old_string 在文件中唯一出现。如果不唯一，编辑失败并提示。

（代码从略：这段是失败时的错误提示，大意是找到了 N 处要替换的匹配，但 replace_all 是 false；想替换所有出现就把 replace_all 设为 true，只想替换一处就提供更多上下文来唯一定位这个实例。）

这个约束的设计哲学是"宁可失败也不猜测"。第一，防止歧义：如果 old_string 是 return null，文件中可能有 5 处 return null。没有唯一性约束，工具只会替换第一个匹配——但模型想替换的可能是第三个。失败并要求模型提供更多上下文（比如包含周围的函数签名），远比猜测性地替换第一个更安全。第二，要求理解上下文：这迫使模型在编辑前真正理解代码结构。模型不能偷懒只提供一个关键词，而是需要提供足够的上下文片段来唯一标识修改点。第三，replace_all 作为显式逃逸阀：当需要重命名变量等批量操作时，模型必须显式把 replace_all 设为 true。这个设计让批量替换成为一个"明确的选择"，而非"意外的后果"。

10.2.6 实现细节：从匹配到写入

引号标准化

文件中可能包含弯引号，英文叫 curly quotes，这种情况在从 Word、Google Docs 或网页复制过来的代码中很常见。但模型输出的始终是直引号，英文叫 straight quotes。如果不做处理，old_string 会因为引号不匹配而查找失败。

Claude Code 在 utils.ts 中实现了一套引号标准化机制。

（代码从略：这段代码定义了 normalizeQuotes() 函数，把左右双弯引号和左右单弯引号统一替换成对应的直引号，用于匹配。）

findActualString() 实现了两阶段匹配策略。

（代码从略：这段代码第一阶段做精确匹配，命中就直接返回；第二阶段把搜索串和文件内容都做引号标准化后重新查找，找到就返回文件中的原始片段以保留弯引号，两种方式都找不到就返回空。）

当通过引号标准化匹配成功后，Claude Code 还会通过 preserveQuoteStyle() 将 new_string 中的直引号转换回弯引号，保持文件的排版一致性。这个函数使用启发式规则判断引号的开闭位置——前面是空白或开括号的是左引号，否则是右引号——并且正确处理缩略语中的撇号（比如 don't）。

除了引号标准化，还有一套反消毒机制：Claude API 会把 function_results、name 这类 XML 标签消毒成 fnr、n 的短形式，模型输出编辑时用的是消毒后的形式。desanitizeMatchString() 在匹配失败时自动还原这些标签。

Diff 生成

getPatchForEdit() 负责将编辑操作转化为结构化的 diff patch。

（代码从略：这段代码逐个应用编辑，每次应用后如果文件内容没有任何变化，就抛出"字符串在文件中找不到"的错误；全部应用完成后，调用 diff 库生成结构化补丁，注意生成前会先把行首 tab 转为空格，用于显示。）

在调用 structuredPatch 之前，内容中的 & 和 $ 字符会先被替换为特殊 token 转义掉，因为 diff 库在处理这些字符时存在 bug。diff 计算后再反转义回来。

删除操作的特殊处理

当 new_string 为空时（即删除操作），applyEditToFile() 有一个贴心的细节：它会检查文件中是否存在 old_string 加一个换行符——如果存在，会连同尾部的换行符一起删除。这防止了删除一行代码后留下一个空行的常见问题。

（代码从略：这段代码区分两种情况——new_string 非空时正常替换、精确执行；删除场景下，如果 old_string 后面紧跟换行符，就把 old_string 连同换行一起删掉，否则仅删除 old_string 本身。）

举个例子：假设文件有三行内容，模型想删除第二行。如果直接把第二行替换为空字符串，结果会在第一行和第三行之间多出一个空行。有了这个处理，实际删除的是第二行连同它后面的换行符，结果干干净净，符合用户预期。

注意这个行为只在 new_string 为空时触发，正常的替换操作不受影响。这属于"默默做正确的事"的设计：用户不必知道这个机制存在，但删除的结果始终符合直觉。

编辑去重

在实际使用中，模型偶尔会因为重试逻辑等原因发送重复的编辑请求。Claude Code 通过 areFileEditsInputsEquivalent() 做语义去重——先比较两组编辑的字面值，字面值不同时再把两组编辑分别应用到当前文件内容，比较最终结果是否一致。

（代码从略：这段代码走两条路径——快速路径直接比较两组编辑的字面值是否完全相同；慢速路径把两组编辑分别应用到原文件内容，比较得到的最终文件是否一致。）

这种语义比较能识别出"输入不同但效果相同"的编辑。例如，两组编辑可能使用了不同长度的 old_string 上下文，但最终修改的内容完全一致——它们会被正确判定为等价，避免重复执行。

10.3 FileWriteTool：全文件写入

FileWriteTool 的定位是创建新文件或完整重写。

（代码从略：这段代码定义了写入工具的输入格式，只有两个字段——文件的绝对路径和完整的文件内容。）

FileWriteTool 的工具描述随 tools 数组一起下发给模型，不是 system prompt，其中的使用指引是：第一，对已有文件，必须先用 Read 工具读取内容，然后编辑。第二，优先使用 Edit 工具修改现有文件，因为它只发送 diff。第三，只在创建新文件或完整重写时使用 Write。第四，永远不要创建文档文件，也就是 md 或 README，除非用户明确要求。第五，避免使用 emoji，除非用户要求。

换行符策略：为什么 Write 始终使用 LF

FileWriteTool 在写入磁盘时始终使用 LF 换行符，不保留原文件的换行风格。源码注释写明：Write 是全内容替换，模型发送的显式换行符就是它的意图，不要改写它们；实现上以 LF 风格调用统一的文本写入函数。

这个决策来自一个真实的 bug 教训。源码注释记录了历史：Previously we preserved the old file's line endings (or sampled the repo via ripgrep for new files), which silently corrupted e.g. bash scripts with \r on Linux when overwriting a CRLF file or when binaries in cwd poisoned the repo sample.

旧版本会保留原文件的换行风格；如果是新文件，会通过 ripgrep 采样仓库中其他文件的换行风格来决定。但这导致了两个问题：第一，在 Linux 上覆盖一个 CRLF 文件时，Write 会给新内容也加上回车符，导致 bash 脚本因为行尾多出回车符而无法执行。第二，当工作目录中有二进制文件时，ripgrep 采样可能将二进制内容误判为 CRLF，污染新文件的换行符。

这与 FileEditTool 形成了刻意的不对称。第一，换行符方面：FileEditTool 保留原文件的换行风格，FileWriteTool 始终使用 LF。第二，编码方面：两者都保留原文件编码。第三，设计原则方面：FileEditTool 是最小变更，只改目标文本；FileWriteTool 是模型意图，内容即真相。

为什么两者不同？FileEditTool 只修改文件的一小部分，保留换行风格是"最小变更"原则的自然延伸。FileWriteTool 替换整个文件，模型发送的内容（包括换行符）代表了完整的意图，不应被工具层面覆写。

编码检测

FileEditTool 的验证管线中包含一个 BOM 检测步骤，BOM 即字节顺序标记，用于正确读取非 UTF-8 编码的文件。

（代码从略：这段代码读取文件的字节缓冲区，如果前两个字节是 UTF-16LE 的字节顺序标记，就按 UTF-16LE 解码，否则按 UTF-8 处理。）

如果文件前两个字节是 UTF-16LE 的字节顺序标记，使用 UTF-16LE 解码；否则默认 UTF-8。完整的 detectEncodingForResolvedPath() 还能识别 UTF-8 自己的字节顺序标记；空文件默认按 UTF-8 而非 ASCII 处理，避免在后续写入 emoji 或中文时出现编码损坏。

安全验证

FileWriteTool 在执行写入前会进行多层安全检查，与 FileEditTool 共享核心验证逻辑。

（代码从略：这段代码依次执行六项检查——团队记忆密钥检查；权限 deny 规则匹配；Windows UNC 路径检查，防止 NTLM 凭据泄露，命中就把文件系统操作交给权限系统处理；文件存在性检查加修改时间统计；读取前置检查，文件没读过就直接拒绝；最后是外部修改检测，读取之后文件被改过也拒绝。）

FileWriteTool 与 FileEditTool 共享同一套 readFileState 缓存机制——对已有文件，必须先读取才能写入。这个约束在代码层面强制执行，而不仅仅是提示词层面的建议。如果文件不存在，即 ENOENT，验证直接通过——这是"创建新文件"的正常场景。

FileEditTool 还额外检查文件大小，上限 1 GiB，防止 V8 和 Bun 约 2 的 30 次方个字符的字符串长度限制导致内存溢出。

（代码从略：这段代码定义了 1 GiB 的最大可编辑文件大小，文件超过这个大小就拒绝编辑，并提示文件过大。）

10.4 编辑前的读取要求

FileEditTool 的工具描述——随 tools 数组一起下发给模型，不是 system prompt——强制要求编辑文件前必须先读取。

（代码从略：这段是工具描述的英文原文，意思是必须在对话中至少用 Read 工具读取一次文件之后才能编辑，尝试编辑没读过的文件会直接报错。）

这不仅是工具描述层面的约束——FileEditTool 的实现中实际检查 readFileState 缓存，如果文件未被读取过，会返回错误，即错误码 6。"先读再改"的高层意图在真实系统提示词 constants/prompts.ts 里也有，措辞较软，只是"read it first"；工具描述则把它落成会硬报错的强约束。

这个设计确保模型了解文件的当前状态，不会基于过时的记忆进行编辑，也能提供正确的 old_string。

没有这个约束会怎样？

两个真实的使用场景可以说明读取前置为什么必要。

场景一是过期记忆。用户在对话的第 3 轮让 Claude 修改 utils.ts 中的 formatDate() 函数。Claude 在第 1 轮读取过这个文件，知道函数签名是 function formatDate(date: Date)。但在第 2 轮中，用户在 IDE 中手动将签名改为带 locale 可选参数的新版本。如果没有读取前置约束，Claude 会基于对话历史中的旧版本生成 old_string——这个字符串在当前文件中已经不存在了，因为多了 locale 参数，编辑会失败。更糟糕的情况是，如果 Claude 使用 FileWriteTool 全文件重写，旧版本的内容会直接覆盖用户刚做的手动修改。

场景二是 isPartialView 的陷阱。CLAUDE.md 和 MEMORY.md 等会被自动注入上下文的记忆文件，注入给模型的内容可能与磁盘不一致，比如剥离了其中的 HTML 注释和 frontmatter，或 MEMORY.md 被截断。这类文件的 readFileState 条目会带 isPartialView 为 true——此时 content 字段存的是磁盘原始内容，而非模型实际看到的版本。如果允许基于部分视图进行编辑，模型看到的内容与缓存里的磁盘原文不一致，old_string 极有可能匹配失败或匹配到错误的位置，因此 Edit 和 Write 都要求对这类条目先做一次真正的 Read。

实现上，它区分了"完全没读"和"读了但只是部分视图"两种情况，对两者都拒绝编辑。

（代码从略：这段代码从 readFileState 缓存取出条目，如果条目不存在，或者标记为部分视图，就返回失败，提示文件尚未读取、请先读再写，错误码为 6。）

并发安全：文件状态缓存

readFileState 是工具上下文中的一个缓存，记录每个文件的读取状态。

（代码从略：这段代码定义了缓存条目的结构——content 存文件内容，用于匹配验证和内容比较；timestamp 存读取时的修改时间，用于外部修改检测；部分读取时还记录起始行 offset 和行数 limit；另有 isPartialView 标记是否为部分视图。）

编辑前的并发检测流程如下。

（代码从略：这段伪代码描述了检测流程——第一步，读取文件当前的修改时间；第二步，与 readFileState 缓存的 timestamp 对比；第三步，修改时间更晚就说明文件被外部修改，返回警告；第四步是 Windows 回退，修改时间不可靠时，比如云同步、杀毒软件等可能触发修改时间变化，就用内容哈希或全文比较作为二次确认。）

这解决了一个常见的竞争条件：用户在 IDE 中编辑文件的同时，Claude Code 也在编辑同一个文件。mtime 检查能捕获这种并发修改，避免覆盖用户的手动改动。

具体实现中，Windows 平台的 mtime 检查有特殊处理。Windows 上 OneDrive 这类云同步和杀毒软件可能在不修改文件内容的情况下更新 mtime，所以检测到 mtime 变化时，如果上次是完整读取而非带 offset 和 limit 的部分读取，会额外比较文件内容——内容相同则认为安全，可以继续编辑。

（代码从略：这段代码先判断上次读取是否完整，即没有指定 offset 和 limit；完整读取且当前内容与缓存内容一致，就认为文件未被改动，否则抛出"文件被意外修改"的错误。）

编辑成功后，readFileState 会立即更新为新的内容和时间戳，防止后续编辑触发误报。这个更新至关重要——如果不更新，模型在同一个 turn 中对同一文件做第二次编辑时，刚才写入产生的新 mtime 会大于旧的 readTimestamp，触发"文件被外部修改"的误报。更新后，后续编辑可以正常进行而不需要重新读取文件。

10.5 多文件编辑协调

需要跨多个文件协调修改时——比如重命名一个被广泛引用的函数——Claude Code 的策略如下。

串行编辑

FileEditTool 未覆写 isConcurrencySafe()，取 buildTool 的默认返回值 false，于是 toolOrchestration.ts 里的调度器 partitionToolCalls 会把它划入串行批次，多个文件编辑操作串行执行；isReadOnly 只用于权限 UI 展示、记忆抽取等场景，不决定并发。串行执行确保三点：第一，不会出现竞争条件；第二，每个编辑基于文件的最新状态；第三，如果中间某个编辑失败，后续编辑不会在错误基础上继续。

原子性考量

单个 FileEditTool 调用是原子的——要么成功替换，要么完全不修改。但跨多个文件的编辑序列不是原子的。如果中间失败，已完成的编辑不会回滚。

这是一个有意的设计权衡：第一，在工具层实现回滚会让复杂度涨一大截；第二，Git 提供了天然的回滚能力，比如 git checkout；第三，模型可以在失败后自主修复。

级联编辑保护

当多个编辑操作在同一文件上依次执行时，有一个微妙的风险：前一个编辑插入的文本可能被后一个编辑意外匹配到。getPatchForEdits() 通过子串检查来防止这种级联错误。

（代码从略：这段代码维护一个已应用新字符串的列表，执行每个编辑前，先去掉 old_string 尾部的换行，再检查它是否是之前任何一个新字符串的子串，是就直接报错；编辑执行后，把新字符串加入列表。）

举个例子：假设编辑 A 将 foo() 替换为 foo()，并在后面加了一条提到 bar() 的注释，编辑 B 想将 bar() 替换为 baz()。如果没有级联保护，编辑 B 的 old_string 即 bar()，会匹配到编辑 A 刚插入注释里的 bar()，导致注释内容被意外改动——这不是模型的意图。有了子串检查，系统会检测到 bar() 是前一个 new_string 的子串，直接报错让模型重新思考编辑策略。

注意检查前会对 old_string 用正则去掉尾部换行，避免因为换行符差异导致的误判。

Worktree 隔离

对于大规模重构，AgentTool 支持 Git Worktree 隔离模式。子 Agent 在独立的 Worktree 中工作，完成后由用户决定是否合并。

（代码从略：这段代码是子 Agent 的配置示例——prompt 写明重构任务，isolation 参数设为 worktree，表示在独立 Worktree 中工作。）

10.6 缩进保持

FileEditTool 的工具描述里有关于缩进的明确指引，这段具体措辞只在工具描述中，system prompt 里没有。

（代码从略：这段英文原文要求，编辑来自 Read 工具输出的文本时，必须原样保留行号前缀之后出现的精确缩进，包括 tab 和空格。）

这特别重要，因为 Read 工具的输出带有行号前缀，也就是 cat -n 格式，模型需要正确区分行号前缀和实际文件内容中的缩进。

10.7 NotebookEditTool：Jupyter 编辑

对于 Jupyter Notebook，也就是 ipynb 文件，Claude Code 提供专门的 NotebookEditTool，它理解 Notebook 的 cell 结构，在 cell 级别精确编辑。

输入 Schema

（代码从略：这段代码定义了 Notebook 编辑工具的输入格式——notebook 文件的绝对路径；目标 cell 的 ID 或 cell-N 格式的索引；新的 cell 内容；cell 类型，插入时必须指定，可选 code 或 markdown；以及编辑模式，可选替换、插入或删除，默认替换。）

工作原理

Jupyter Notebook 就是一个 JSON 文件，核心结构是 cells 数组。每个 cell 包含 cell_type、source、metadata，以及 code cell 特有的 outputs 和 execution_count。

NotebookEditTool 有三种编辑模式。第一，replace，替换指定 cell 的 source 内容。对 code cell，会同时把 execution_count 重置为 null 并清空 outputs——因为源代码已变，旧的输出不再有效。第二，insert，在指定 cell 之后插入新 cell。如果不指定 cell_id，就在开头插入。对于 nbformat 4.5 及以上版本的 notebook，会自动生成随机 cell ID。第三，delete，删除指定 cell，通过在 cells 数组上执行切片删除实现。

Cell 定位

Cell 的定位支持两种方式。第一种是原生 cell ID，直接使用 notebook 中每个 cell 的 id 字段。第二种是索引格式，即 cell-N 格式，比如 cell-0、cell-3，由 parseCellId() 解析为数字索引。

边界情况处理

NotebookEditTool 的实现处理了几个边界情况。

Replace 自动转 Insert：当 edit_mode 为 replace 但 cellIndex 等于 cells.length，即指向末尾之后时，自动降级为 insert 模式。这容错了模型在计算 cell 索引时的常见 off-by-one 错误。

Cell ID 版本兼容：只有 nbformat 4.5 及以上版本的 notebook 才支持 cell ID。对于旧版格式，插入新 cell 时不会生成 id 字段，避免写入不被识别的字段导致兼容性问题。新 cell ID 用随机数生成的随机字符串。

非缓存 JSON 解析：call() 方法中使用非缓存的 jsonParse() 而非 safeParseJSON() 来解析 notebook 内容。这是因为后续会直接修改解析出的对象，比如用切片删除 cell、直接改写目标 cell 的内容；如果使用缓存版本，修改会污染缓存，导致 validateInput() 和后续调用拿到已被篡改的对象。

与 FileEditTool 的验证差异

虽然 NotebookEditTool 共享了部分安全机制，但验证管线与 FileEditTool 差别不小。第一，读取前置方面：FileEditTool 要求完整读取，拒绝部分视图；NotebookEditTool 只要求读取过，不检查是否部分视图。第二，字符串匹配方面：FileEditTool 做精确匹配、引号标准化加反消毒；NotebookEditTool 没有这一层，按 cell 定位，不做字符串匹配。第三，唯一性约束方面：FileEditTool 要求 old_string 必须唯一；NotebookEditTool 不适用，cell 的 ID 或索引天然唯一。第四，文件大小限制方面：FileEditTool 有 1 GiB 上限；NotebookEditTool 无限制。第五，编辑前密钥检查方面：FileEditTool 检查 new_string 是否包含密钥；NotebookEditTool 不检查。第六，配置文件保护方面：FileEditTool 对 settings.json 做 Schema 校验；NotebookEditTool 不适用。

这种差异是合理的：Notebook 的 cell 结构提供了天然的定位机制，即 cell ID 或索引，不需要 FileEditTool 那套基于字符串匹配的复杂验证。但这也意味着 NotebookEditTool 的安全防护层次更少——它更依赖 notebook 本身的结构化格式来保证编辑不出错。

权限与安全

NotebookEditTool 共享与 FileEditTool 相同的核心安全机制：同样要求先读取 notebook 文件才能编辑，与 FileEditTool、FileWriteTool 一致；同样通过 mtime 对比检测文件是否被外部修改；同样拦截 Windows UNC 路径。权限分组上，在 acceptEdits 权限模式下它与 FileEditTool 一样自动批准，无需用户确认。写入后同样更新 readFileState 缓存，保持与其他编辑工具的一致性。

10.8 原子写入与 LSP 集成

FileEditTool 的 call() 方法实现了一个完整的编辑执行管线，从文件读取到写入后的各种副作用。

（代码从略：这段代码是编辑执行的主流程。写入前做三件可异步的准备——发现技能目录、确保父目录存在、做文件历史备份。随后进入临界区，同步读取文件并带上编码和换行符元数据，做基于修改时间的过时检测，再做引号标准化、查找匹配、生成补丁，最后写入磁盘并保持原始编码和换行符。写入后处理副作用——通知 LSP 服务器文件的修改和保存，通知 VSCode 以更新 diff 视图，更新 readFileState 缓存，并统计变更行数、记录文件操作日志。）

几个关键设计点：

临界区最小化：步骤 4 到 8 之间刻意避免任何 await 异步操作。注释中明确写道：Please avoid async operations between here and writing to disk to preserve atomicity。原因在于 JavaScript 和 TypeScript 是单线程但基于事件循环的——每个 await 都是一个让出控制权的点。如果步骤 5 的过时检测和步骤 8 的写入磁盘之间有 await，linter 自动修复、IDE 保存这类异步操作就可能趁这个间隙修改文件，写入时把外部修改覆盖掉。将文件历史备份和目录创建等可异步的操作提前到临界区之外，确保检测和写入之间没有断点。

编码与换行符的完整往返管线：FileEditTool 对文件编码和换行符的处理遵循"读什么写什么"原则，整个管线分为三个阶段。第一阶段是读取阶段，调用 readFileSyncWithMetadata()。它先检测编码，通过 BOM 判断：见到 UTF-16LE 的字节顺序标记就按 UTF-16LE，见到 UTF-8 的字节顺序标记就按带 BOM 的 UTF-8，默认 UTF-8。再检测换行符：扫描原始内容前 4096 个码元，统计 CRLF 和独立 LF 的数量，多数投票决定换行风格。最后规范化内容，把所有回车加换行统一为单个换行，使内部处理全程基于 LF。第二阶段是处理阶段：所有字符串匹配、替换、diff 生成都基于 LF 规范化后的内容，这简化了 old_string 的匹配逻辑——模型不需要关心目标文件是 LF 还是 CRLF。第三阶段是写入阶段，调用 writeTextContent()：如果检测到的原始换行符是 CRLF，先把内容中的换行全部替换回回车加换行，再使用检测到的原始编码写入磁盘。

这意味着编辑一个 UTF-16LE 加 CRLF 的文件，即 Windows 上的旧式文本文件，内部全程用 UTF-8 加 LF 处理，写入时恢复为 UTF-16LE 加 CRLF——文件的编码和换行风格完全不变。

LSP 通知：编辑完成后立即通知 LSP 服务器，分为两步——changeFile() 对应 didChange 通知，告知内容已修改；saveFile() 对应 didSave 通知，触发 TypeScript server 等语言服务器的诊断更新。这些通知都是 fire-and-forget，异常只做日志，不阻塞编辑返回。同时会清除该文件之前已交付的诊断信息，即 clearDeliveredDiagnosticsForFile，确保新诊断不会被去重过滤。

文件历史备份：fileHistoryTrackEdit() 在写入前把文件原始内容备份一份。备份文件按文件路径的哈希命名，取路径的 sha256 哈希前 16 位，再加 @v1 后缀——注意哈希对象是路径不是内容。去重粒度是"当前 snapshot 内同一文件只备份一次"：第二次 trackEdit 直接跳过，避免用编辑后的内容覆写掉 v1 备份；真正的内容级去重发生在 makeSnapshot 阶段，靠读回文件逐字节比较，同样不走哈希。备份用 copyFile 复制而非把大文件读进 JS 堆，避免内存溢出，存放在用户目录下 .claude 的 file-history 会话目录中。备份是幂等的，即使后续的过时检测失败、编辑中止，多出的备份也不会影响状态一致性。另外，FileEditTool.ts 源码里"按内容哈希为键"的注释与实现不符，实际是路径哈希加 snapshot 级去重。

技能目录发现：当编辑的文件位于某个技能目录中时，discoverSkillDirsForPaths() 会识别出该目录，并触发动态技能加载。此外 activateConditionalSkillsForPaths() 会激活路径匹配的条件技能。这使得编辑技能文件时，新技能可以立即被发现和使用。

10.9 Diff 渲染

编辑完成后，Claude Code 需要在终端中向用户展示变更内容。这由 StructuredDiff 组件负责，位于 src/components/StructuredDiff.tsx。

数据结构

Diff 渲染的核心数据来自 diff 库的 StructuredPatchHunk。

（代码从略：这段代码定义了补丁块的结构——oldStart 和 newStart 分别是原文件与新文件的起始行号，oldLines 和 newLines 分别是两边涉及的行数，lines 是 diff 行列表，每行用加号、减号或空格前缀表示增加、删除或不变。）

关键常量：diff 上下文行数是 3 行，diff 计算超时是 5 秒。

行号调整

有时 getPatchForDisplay 拿到的只是文件片段，比如 readEditContext 提供的局部内容，此时 hunk 的行号相对于片段起始位置。adjustHunkLineNumbers() 将其转换为文件级别的绝对行号。

（代码从略：这段代码把每个补丁块的 oldStart 和 newStart 都加上偏移量，得到文件级别的绝对行号。）

语法高亮

StructuredDiff 组件使用 color-diff 原生模块渲染语法高亮，这是一个 Rust NAPI 模块。整个渲染流程有多层缓存优化。第一层是 WeakMap 缓存，以补丁块对象引用为键缓存渲染结果，当 hunk 对象被垃圾回收时，缓存自动释放。第二层是参数化缓存键，缓存键包含主题、宽度、暗淡标记、行号槽宽度，以及用于 shebang 检测的首行和用于语言检测的文件路径，确保相同参数命中缓存。第三层是缓存上限，每个 hunk 最多保留 4 个缓存条目，覆盖正常使用场景中的宽度变化，超出后清空重建。

（代码从略：这段代码用 WeakMap 以补丁块对象为键建立渲染缓存，缓存键把主题、宽度、暗淡、行号槽宽度、首行和文件路径等所有影响渲染的参数编码进去。）

终端渲染布局

在全屏模式下，diff 使用双列布局——行号槽列放行号和加减号标记，内容列放正文，两列分离。

（代码从略：这段代码根据补丁块中的最大行号位数计算行号槽宽度——取原文件和新文件各自末行行号的较大者，在它的位数上加 3，即 1 个标记符加两个间隔空格。）

行号槽列用防选中组件包裹，用户在终端中选择复制 diff 内容时就不会带上行号——一个小而实在的体验优化。sliceAnsi 函数在切割 ANSI 彩色文本时保持转义序列的完整性，确保颜色不会因为列分割而错乱。

当行号槽宽度超过终端总宽度时，即极窄终端场景，会自动回退到单列渲染，由 Rust 模块处理自动换行。如果原生模块不可用或语法高亮被禁用，则回退到 StructuredDiffFallback 组件进行纯文本渲染。

10.10 与工具系统的整合

编辑工具在工具系统中的位置是这样的。首先，模型决策并选择工具：修改已有文件时选 FileEditTool，走 search-and-replace 路线；创建新文件时选 FileWriteTool，做全文件写入；遇到 Notebook 就选 NotebookEditTool，做 cell 级编辑。然后，三种工具都进入权限检查。最后，在 acceptEdits 模式下自动批准，在 default 模式下需要用户确认。

FileEditTool 与 FileWriteTool 都不覆写 isReadOnly 和 isDestructive，两者均取 buildTool 默认值，即都是 false——它们的破坏性差异体现在"改局部还是覆写全文"的策略上，而非这两个布尔标志。

在 acceptEdits 权限模式下，编辑类工具自动批准，无需用户确认——信任度高的项目因此不必被逐次确认打断。

10.11 关键设计洞察

第一，低破坏性是核心原则：search-and-replace 不是因为简单才被选择，而是因为它对代码库的影响最小。第二，失败比静默错误更好：唯一性约束确保模型不会在歧义场景下做出错误编辑。第三，读取前置是安全网：强制读取确保模型基于最新状态做编辑。第四，replace_all 的克制使用：默认 false，只在明确的批量操作场景下启用。第五，Git 是终极回滚机制：不需要在编辑工具层面实现复杂的事务或回滚。第六，引号标准化是现实主义：处理真实世界中从各种来源复制粘贴的代码，而不是假设所有文件都完美规范。第七，LSP 集成让 IDE 实时响应：编辑后立即通知语言服务器，用户无需等待就能看到最新的诊断信息。第八，文件历史提供额外安全网：按路径哈希命名、snapshot 级去重的幂等备份，在 Git 之外多一层保护。第九，输入预处理无声但关键：尾部空白裁剪和 API 反消毒在验证前静默修正模型输出的常见瑕疵，用户和模型都不需要感知这个过程。第十，Edit 与 Write 的换行符不对称是刻意的：Edit 保留原始换行风格，遵循最小变更原则；Write 始终使用 LF，遵循模型意图原则，两者在各自的场景下都是正确的选择。第十一，级联保护防止自引用编辑：多步编辑中的子串检查确保后续编辑不会意外修改前一步刚插入的文本，将一类难以调试的 bug 拦截在源头。

这个编辑策略的精髓可以用一句话概括：宁可编辑失败让模型重试，也不要静默地写入错误内容。

动手实践：在 claude-code-from-scratch 的 src/tools.ts 中，edit_file 工具实现了简化版的 search-and-replace 策略。尝试运行 npm start 让 Agent 编辑一个文件，观察唯一性约束在实际中如何工作。

上一章：Plan 模式。下一章：任务管理系统。
