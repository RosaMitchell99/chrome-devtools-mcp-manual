章节：01-tab-and-context.md（浏览器标签页与上下文管理）
相关源码位置（主要文件）
McpContext.ts
Page 列表管理：#pages, createPagesSnapshot(), getPages(), getSelectedPage(), setSelectedPageIdx(idx)。
页面生命周期操作：newPage(), closePage(pageIdx)。
dialog 管理：#dialog、#dialogHandler、getDialog(), clearDialog()、handleDialog 在 pages.ts 中调用。
资源与仿真状态：#networkConditionsMap、#cpuThrottlingRateMap、setNetworkConditions(), setCpuThrottlingRate(), #updateSelectedPageTimeouts()（管理超时/节流策略）。
snapshot 与元素定位相关（见 snapshot 章节）：createTextSnapshot(), getElementByUid(), getAXNodeByUid()。
PageCollector.ts
PageCollector 与 NetworkCollector 的实现（用于收集 console / network / page events），关键点：storage（WeakMap<Page, Array>）、id 生成器（stableIdSymbol）、init() 中对新 page 的监听。
pages.ts
list_pages、select_page、new_page、close_page、navigate_page_history、resize_page、handle_dialog（tool 定义层） —— 这些是暴露给 MCP 客户端的“页与上下文”操作。
src/formatters/snapshotFormatter.ts（部分与页面快照展示有关，见 snapshot 章节）
在文档中可以讲的点
McpContext 是“上下文层”的核心：它封装了 Browser 与 Page 的状态（哪个 tab 被选中、当前 snapshot、网络/CPU 仿真等），文档应把它描述为“宿主应用的运行时状态管理器”。
PageCollector 的设计：弱引用（WeakMap）避免内存泄漏，给每条收集的数据分配稳定 id（stableIdSymbol），并在 main-frame 导航时清理历史，说明这对 network/listing 的“自上次导航以来”语义很重要。
选择页面的时序与副作用：setSelectedPageIdx 会解绑旧 page 的 dialog 监听并绑定新 page，要注意 dialog 状态、timeout 更新等副作用。
超时与仿真策略：CPU / network 仿真如何通过 multiplier 影响 page.setDefaultTimeout 和 setDefaultNavigationTimeout（#updateSelectedPageTimeouts）。
可在章中引用/展示的函数（示例）
createPagesSnapshot(), setSelectedPageIdx(), newPage(), closePage()
PageCollector#getById / NetworkCollector.cleanupAfterNavigation（解释“自上次导航保留请求”）
测试参考（可作为示例的单元测试）
tests/tools/pages.test.ts
tests/PageCollector.test.ts
边界/注意事项（需要在文档中强调）
不能关闭最后一个 tab（CLOSE_PAGE_ERROR），会在 tools 层捕获并将友好消息返回。
getSelectedPage() 在 page 已关闭时会抛错（并指示调用 listPages）。
选择页面时需要小心 dialog 的存在（可能阻塞交互）。
章节：02-snapshot-and-aria.md（快照机制、ARIA 语义）
相关源码位置
McpContext.ts
createTextSnapshot(verbose?)：通过 page.accessibility.snapshot(...) 收集 AX 树，给每个节点分配 id（snapshotId_counter），并把 node 存入 idToNode map，后续通过 uid（格式 snapshotId_index）可反查。
getAXNodeByUid(uid), getElementByUid(uid)：如何把 snapshot uid 映射回 ElementHandle 并获取 handle。
snapshot.ts
take_snapshot tool：设置 response.setIncludeSnapshot(true)（工具层触发 snapshot 的生成和包含）。
wait_for tool：使用 Locator.race 查找 aria / text，并在发现后设置包含 snapshot。
snapshotFormatter.ts
formatA11ySnapshot(serializedAXNodeRoot)：用于把序列化的 a11y 树格式化为人可读的文本（用于响应体）。
tests/
tests/snapshot.test.ts（查看测试示例及期望输出）
在文档中可以讲的点
为什么用 Accessibility (a11y) tree：a11y tree 把页面元素按语义组织（role、name、value、states），更适合给 LLM 理解页面结构（而非 raw DOM）。
uid 的生成与有效性：snapshotId 与局部 id 的组合确保 snapshot 一致性，说明 stale uid 的错误情形（McpContext.getElementByUid 会检测 snapshotId 是否与当前 snapshot 匹配并报错）。
从 snapshot 到实际元素：show 代码如何用 node.elementHandle() 取得 ElementHandle（并注意对 handle 的 dispose）。
格式化输出：formatA11ySnapshot 如何把属性（role, name, states）转换为文本，便于模型或用户阅读。
示例/演示
展示一个简单的 a11y 输出（formatA11ySnapshot 的输出示例），并示范如何用 uid 点击元素（结合 click 工具）。
测试参考
tests/snapshot.test.ts
边界/注意事项
snapshot 可能不包含所有属性（option 的 value 在 AX node 中可能缺失，需要额外处理，见 createTextSnapshot 中 option 特殊处理）。
uid 过期导致的错误处理路径（stale snapshot 报错）。
elementHandle 可能不可用（在拿到 handle 后 element 已被移除）。
章节：03-tools-and-schema.md（MCP 工具定义、schema）
相关源码位置
ToolDefinition.ts
defineTool<T>、ToolDefinition 接口、Request/Response 抽象、Context 类型（说明工具能调用的上下文 API）。
timeoutSchema：公用的 timeout schema 设计（示例）。
CLOSE_PAGE_ERROR 常量。
各工具实现目录 src/tools/*.ts（每个文件即一组工具）
pages.ts（导航与页面管理）
input.ts（click/hover/fill/drag/upload_file/fill_form）
snapshot.ts（take_snapshot, wait_for）
performance.ts（performance_start_trace/performance_stop_trace/performance_analyze_insight）
network.ts（list_network_requests/get_network_request）
emulation.ts（emulate_cpu/emulate_network/resize_page）
console.ts / script.ts / screenshot.ts 等
zod 用来定义 schema（在每个 tool 文件中使用 z）
main.ts 中的 registerTool：如何把 ToolDefinition 注册到 MCP server（把 tool.schema 作为 MCP inputSchema 暴露）
在文档中可以讲的点
defineTool 的约定：工具由 name/description/annotations/schema/handler 构成；annotations 指明 category、readOnlyHint（用于文档或 UI 提示）。
使用 zod 定义参数 schema：示例展示（比如 navigate_page 的 url 与 timeout），解释 schema 的可读性以及客户端如何根据 schema 生成 UI/验证。
Response 接口（McpResponse）的责任：appendResponseLine、setIncludeSnapshot、attachImage、setIncludeNetworkRequests 等，说明工具 handler 不直接拼文本，而是通过 Response 指示要包含哪些数据，最终由 McpResponse.format 统一渲染并返回。
工具实现模式：先在 handler 中调用 context（McpContext）执行动作，再调用 response.setIncludeXXX() 或 response.appendResponseLine()。强调 waitForEventsAfterAction 的使用（保证在 action 后等待 network/console 等事件稳定）。
分类/注解用法（ToolCategories）便于把工具分块文档化。
示例/可复制片段
以 click/fill/new_page 为示例，展示 zod schema 与 handler 的典型样式。
测试参考
tests/tools/input.test.ts, tests/tools/pages.test.ts, tests/tools/network.test.ts, tests/tools/performance.test.ts
边界/注意事项
工具 handler 里的异常处理与用户友好消息（如 closePage 捕获 CLOSE_PAGE_ERROR 并 appendResponseLine）。
handler 中对 ElementHandle 的 dispose 要注意（防止内存泄漏）。
Response 和 Context 的分工（工具作者只需关心 handler 逻辑，不直接拼最终字符串）。
章节：04-agent-communication.md（LLM 与 MCP Server 之间的通信、调用关系与安全）
相关源码位置
main.ts
MCP Server 初始化：new McpServer(...), transport 选择（StdioServerTransport）并 connect。
registerTool(tool): how incoming params -> getContext() -> tool.handler(...) -> response.handle(...) -> 返回 content（CallToolResult）。
Mutex（toolMutex）用于串行化工具调用（防止竞态），注意对并发的限制与理由。
McpResponse.ts
McpResponse.handle(toolName, context): 在工具执行完后，收集 snapshot/pages/network/console、并格式化（format()）返回 TextContent/ImageContent 给 MCP 客户端。
format 里会把 includePages/includeSnapshot/network/console 等扩展到最终文本/图片输出。
src/browser.ts / cli.ts / ensureBrowserLaunched (main.ts 调用)
浏览器启动/连接的实现（确保 server 能够控制 Chrome），可讲述如何配置 headless/isolation/browserUrl 等参数。
src/tools/ToolDefinition.ts（定义 Input/Response/Context contract，便于解释 agent 层如何与工具协作）
src/trace-processing/*（performance trace 解析发送回 agent 的实现）
在文档中可以讲的点（通信与安全）
从调用到响应的完整链路（客户端 -> MCP transport -> McpServer -> registerTool wrapper -> getContext() -> tool.handler -> McpResponse.handle -> 返回 content）：可以配图（时序图）。这段在 main.ts 的 registerTool 中实现，适合截取并解释每一步的职责与错误分支（try/catch）。
传输层与部署：默认使用 StdioServerTransport（本地进程通信），但 MCP 可以使用其他 transport（SSE/WebSocket 等）。说明本实现如何通过 stdin/stdout 与客户端通信（安全边界提示）。
并发与互斥：toolMutex 的作用，解释为什么需要串行化（Puppeteer/Chrome 实例与会话状态的共享冲突）以及可能的扩展（更细粒度锁、每页锁等）。
响应的结构化输出：McpResponse 如何把文本、图片、network/console/snapshot 等统一封装返回给客户端，agent 如何消费（给出示例返回片段）。
安全建议：main.ts 启动日志中的免责声明、如何通过 --isolated（创建独立 user-data-dir）、headless/isolation、以及建议的认证/白名单策略（文档里做建议，而非在代码中实现）。
示例/可复现流程
读者演示：通过 CLI 启动 server，使用模型发起 navigate_page，然后用 list_network_requests / performance_start_trace 查看输出；把 main.ts registerTool 的代码片段作为“调用链示意”。
测试/集成参考
tests/index.test.ts, tests/cli.test.ts（演示主流程的一些集成测试）
边界/注意事项
工具返回的文本是怎样被组装（appendResponseLine vs includeSnapshot vs attachImage），在 agent 端如何解析 content 数组（text + image）并展示。
错误处理：registerTool 中的 try/catch 对工具异常的处理（将 error 转化为 text response 或抛出）与日志记录。
更合理的“分模块”介绍方式（建议） 为了让读者更系统地把“设计 + 实现”结合起来，我建议把源码解读分成下面几个模块（每个模块在文档中都配上关键文件和一个小图）：
核心运行时 / 启动（Core runtime）
主要文件：src/main.ts、src/cli.ts、src/browser.ts、src/logger.ts
介绍：MCP server 启动、transport、browser 启动/连接、工具注册流程、并发控制（Mutex）。
上下文与收集（Context & Collectors）
McpResponse.ts
介绍：页面/快照/网络/控制台收集，snapshot/uid 映射，McpResponse 的聚合与格式化。
工具层（Tools & schema）
主要文件：src/tools/*.ts、src/tools/ToolDefinition.ts、src/tools/categories.ts
介绍：工具如何定义（zod schema）、handler 模式、如何扩展工具（新增 tool 的步骤）。
格式化与输出（Formatters）
主要文件：src/formatters/*.ts（snapshotFormatter, networkFormatter, consoleFormatter）
介绍：如何把采集的数据渲染为文本/图片供模型或用户阅读。
性能与 trace（Trace processing）
performance.ts
介绍：如何启动/停止 trace、解析 trace、生成 insight。
测试与示例（Tests & Examples）
主要文件：tests/**/，特别是 tests/tools/.test.ts、tests/snapshot.test.ts、tests/performance.test.ts
介绍：测试如何覆盖工具行为，哪些测试可以直接用作文档中的示例输出。
文档中推荐加入的附加内容（提高可读性）
时序图（建议放到 04-agent-communication.md）：
client -> main.registerTool -> getContext -> tool.handler -> response.handle -> 返回 content（图片/文本）。我可以为你画好 mermaid 格式的 sequenceDiagram 并插入文档。
快照示例（02-snapshot-and-aria.md）：
formatA11ySnapshot 的输出片段 + 对应 uid，配合 click 示例（“用 uid 点击元素”）做 end-to-end 示例。
schema 示例表（03-tools-and-schema.md）：
列举若干工具的 zod schema（navigate_page / fill / upload_file），并解释字段含义、validation、timeoutSchema 的用途。
关键代码摘取（保持最小片段）：
registerTool 的核心流程（main.ts）和 McpResponse.format 的整体结构（显示最终返回格式），便于读者把“请求 -> 执行 -> 返回”全貌串起来。
引用测试用例输出（Tests）
在每章末尾引用相关 test 文件里 assert 的示例输出，便于读者验证与复现。
我注意到的实现细节与可在文档里强调的 engineering 注意点
PageCollector 使用 WeakMap 和就地清空数组（collection.length = 0），这是为减少重复分配并保留引用（性能优化），值得在文档中注释为何这么做。
UID 的语义设计（snapshotId_index）是避免 stale id 冲突的简单而有效的方案：应在文档里强调取用 snapshot 前后的一致性检查（getElementByUid 会检查 snapshotId）。
waitForEventsAfterAction 的使用模式（在执行点击 / 填入 / 拖拽时把“动作”包裹在此函数里），这一点对保证测试稳定性与避免 race condition 很重要。
调试与安全：McpResponse 会把很多内部信息输出（console / network / snapshot），文档应明确安全风险与最佳实践（不要在生产环境直接暴露未授权的 MCP server）。
重要测试文件清单（便于你引用）
tests/tools/input.test.ts
tests/tools/pages.test.ts
tests/tools/network.test.ts
tests/tools/performance.test.ts
tests/snapshot.test.ts
tests/PageCollector.test.ts
tests/index.test.ts / tests/cli.test.ts（启动/集成层）
后续我可以为你做的具体动作（选一个）
A — 我把上面每章的“源码映射 + 可写点”整理成四个完整的 md 段落，直接插入对应的章节文件（01-tab-and-context.md、02-snapshot-and-aria.md、03-tools-and-schema.md、04-agent-communication.md）。
B — 我仅生成一份可复制的 Markdown 报告供你手动粘贴（不修改仓库）。
C — 我为每章生成一张 mermaid 时序图和一个简短示例（代码片段 + 预期输出），并把它插入到相应 md 文件。
D — 先不改文件，继续深入某些源码（例如把 trace-processing/parse.ts 和 formatters/networkFormatter.ts 也扒一遍）然后把更详细的实现解读写入文档。
请选择你要我接着做的操作（或告诉我你想先看哪个章节的示例草稿），我会马上开始并把变更直接写入文件（若你选 A 或 C）。