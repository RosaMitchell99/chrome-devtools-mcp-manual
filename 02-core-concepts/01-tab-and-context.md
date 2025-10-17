# 第1节 浏览器标签页与上下文管理
## 1.1 概述
在浏览器自动化和测试框架中，标签页与上下文管理是至关重要的组成部分。每个标签页代表一个浏览器的实例，它包含一个独立的页面和相关的操作上下文。MCP 协议的设计目的是通过管理浏览器标签页和上下文的状态，来提供精确且可控的页面交互和测试环境。本节将重点探讨Chrome DevTools MCP如何管理标签页及其生命周期，以及如何实现对页面的选择、创建、关闭等基本操作（以便更高层次的工具调用）。  

## 1.2 McpContext：运行时状态管理类
### 功能概述
src/McpContext.ts 文件实现了整个浏览器上下文管理的核心类 `McpContext`。通过设置类成员和方法，它封装了浏览器与页面的状态，并封装 Puppeteer 的 API（低级浏览器操作），使其与 MCP 协议兼容。它不仅跟踪哪些标签页处于活动状态，还管理页面的生命周期和仿真设置，完整地描述了宿主（host）端的运行时状态，供 MCP 工具使用。

### 调用逻辑
在主逻辑 `main.ts` 中的 `getContext()` 函数会创建、缓存 `McpContext` 实例，并传入下游工具。而这一调用又由 `register_tool()`（或类似注册函数）完成，说明在调用工具之前，系统已经创建并初始化了 `McpContext` 实例。

### 设计与实现详述

#### McpContext — 运行时上下文与浏览器封装

##### 概述
`McpContext` 是项目中连接 MCP（Model Context Protocol）工具层与浏览器（通过 `puppeteer-core`）之间的关键运行时封装。它保存长期运行的状态（浏览器里的 pages 列表、当前选中页面索引、最近一次无障碍文本快照、收集器、trace 存储等），并为上层工具提供一组可复用的低级能力（创建/关闭页面、截取快照、获取元素句柄、设置网络/CPU 节流、记录 trace、文件保存等）。

本节将解释 `McpContext` 的设计职责、内部数据结构、关键方法的行为与调用语义，以及在工具层（`tools/*.ts`）如何正确使用它。

##### 设计契约（短小“合同”）
- 输入：调用方（工具层）传入 puppeteer 的 `Page`、URL、参数（例如节流率、网络档位）等；`McpContext` 通过构造函数获得 `Browser` 和 `logger`。
- 输出：方法通常返回 puppeteer 对象（如 `Page`、`ElementHandle`）、简单结构（`TextSnapshot`）或 `void`；另外 `McpContext` 也通过 collectors 暴露网络/console 数据供工具读取。
- 错误模式：`McpContext` 对非法调用会抛出错误（例如没有选中页面、快照过期、关闭最后一个页面禁止等），上层工具负责捕获并翻译为用户友好信息。
- 成功标准：对页面的状态变更必须同步更新 `#pages`、collector 注册/注销，并保证相关超时/节流参数生效以便后续等待/动作使用。

##### 关键常量与辅助函数
- `DEFAULT_TIMEOUT = 5000`：默认等待超时（ms）。
- `NAVIGATION_TIMEOUT = 10000`：导航等待超时（ms）。
- `getNetworkMultiplierFromString(condition: string | null)`：根据 puppeteer 预设网络档位（如 'Fast 4G'、'Slow 3G'）返回网络放大系数（用于调整导航超时）。
- `getExtensionFromMimeType(mimeType: string)`：将图片 MIME 映射到文件扩展名（png、jpeg、webp），找不到映射会抛错。

##### 内部状态（私有字段，说明其目的）
- `browser: Browser`：注入的 puppeteer `Browser` 实例。
- `logger: Debugger`：错误与调试日志记录器。
- `#pages: Page[]`：当前已打开的页面快照（由 `createPagesSnapshot()` 填充）。
- `#selectedPageIdx: number`：当前选中页面在 `#pages` 的索引。
- `#textSnapshot: TextSnapshot | null`：最近一次通过 accessibility API 得到的树状文本快照（带 id 到节点的映射）。
- `#networkCollector: NetworkCollector`：网络事件收集器，按 page 分组并分配稳定 id。
- `#consoleCollector: PageCollector<ConsoleMessage | Error>`：console / pageerror 收集器。
- `#isRunningTrace: boolean`：是否正在录制性能 trace。
- `#networkConditionsMap: WeakMap<Page, string>`：针对每个 page 的网络条件配置（可为空）。
- `#cpuThrottlingRateMap: WeakMap<Page, number>`：每个 page 的 CPU 节流倍率（默认 1）。
- `#dialog?: Dialog`：当前捕获的对话框（由 dialog 事件处理器持有）。
- `#nextSnapshotId: number`：为 accessibility snapshot 中的每个节点生成 ID 的计数器前缀。
- `#traceResults: TraceResult[]`：存储解析后的 trace 结果供后续查询/分析。

##### 初始化与生命周期
- 构造函数私有，使用 `static async from(browser, logger)` 工厂方法创建实例并执行 `#init()`。
- `#init()` 做了三件事：
  1. 调用 `createPagesSnapshot()` 以加载当前 `browser.pages()` 到 `#pages`；
  2. 将 `selectedPageIdx` 设为 0；
  3. 初始化 `#networkCollector` 与 `#consoleCollector`（注册监听器并开始收集）。
- 设计上上下文是与 `Browser` 生命周期绑定的：当 browser 新开 page、关闭 page 时，`McpContext` 通过方法被动更新或由工具调用相应方法（如 `newPage()`、`createPagesSnapshot()`）。

##### 主要方法详解（按功能分组）

###### 页面管理
- `async createPagesSnapshot(): Promise<Page[]>`
  - 描述：从 `browser.pages()` 读取当前页面列表并把结果写入 `#pages`。
  - 返回值：更新后的 `Page[]`。
  - 使用场景：在外层工具需要刷新页面列表（例如在 new_page 后）时调用，确保 `#pages` 与实际浏览器状态一致。

- `getPages(): Page[]`
  - 描述：返回当前 `#pages` 的引用（只读语义，调用方不应修改数组）。

- `getSelectedPage(): Page`
  - 描述：返回当前选中页面。如果没有选中或选中页面已关闭，会抛出错误（错误信息里会建议调用 `listPages`）。
  - 边界：始终检查 `page.isClosed()` 并以友好信息抛出。

- `getPageByIdx(idx: number): Page`
  - 描述：直接按索引返回 page，若不存在则抛错。

- `getSelectedPageIdx(): number`
  - 简单返回当前索引。

- `async newPage(): Promise<Page>`
  - 描述：封装 `browser.newPage()` 的逻辑，同时：
    - 调用 `createPagesSnapshot()` 更新 `#pages`；
    - 把新创建的 page 设为选中页（通过设置索引）；
    - 把 page 注册到 `#networkCollector` 和 `#consoleCollector`。
  - 语义：工具层调用 `context.newPage()` 就能得到完成注册的 page。

- `async closePage(pageIdx: number): Promise<void>`
  - 描述：关闭 `#pages[pageIdx]` 对应的 page，但含有保护逻辑：如果当前只剩一个 page，会抛出 `CLOSE_PAGE_ERROR`（上层工具会捕获并产出友好报错）。
  - 额外行为：设置 selected index 为 0（默认切回第一个页面），然后调用 `page.close({ runBeforeUnload: false })`。
  - 注意：收集器会在 page 被 close 时自动淘汰相关监听器（collector 设计负责清理）。

- `setSelectedPageIdx(idx: number): void`
  - 描述：切换选中页面。实现细节：
    - 从旧 page 上移除 dialog 监听器；
    - 把 `#selectedPageIdx` 更新为 `idx`；
    - 给新 page 添加 dialog 监听器；
    - 调用 `#updateSelectedPageTimeouts()` 以根据 CPU/网络节流调整超时设置。

###### 对话框（Dialog）支持
- `#dialogHandler`：内部函数绑定到 page 的 `dialog` 事件，用来暂存最近的对话框实例到 `#dialog`。
- `getDialog(): Dialog | undefined`：返回当前 dialog。
- `clearDialog()`：清空保存的 dialog 引用（工具层在处理完后应调用以避免悬留）。

###### 网络 / 控制台收集器访问
- `getNetworkRequests(): HTTPRequest[]`：返回针对当前 selected page 的 network 请求（由 `#networkCollector` 存取）。
- `getNetworkRequestById(reqid: number): HTTPRequest`：按 collector id 检索某条请求。
- `getConsoleData(): Array<ConsoleMessage | Error>`：返回 console / pageerror 收集结果。
- `getNetworkRequestStableId(request: HTTPRequest): number`：返回 collector 分配给该请求的稳定 id，用于在 `McpResponse` 中引用请求。

###### 无障碍文本快照（Accessibility Snapshot）
- 数据结构：
  - `TextSnapshotNode`：继承 `SerializedAXNode`，增加 `id: string` 和 `children: TextSnapshotNode[]`。
  - `TextSnapshot`：{ root: TextSnapshotNode, idToNode: Map<string, TextSnapshotNode>, snapshotId: string }
- `async createTextSnapshot(verbose = false): Promise<void>`
  - 描述：对当前 selected page 调用 `page.accessibility.snapshot({ includeIframes: true, interestingOnly: !verbose })`，遍历得到的树为每个节点分配唯一 id（格式 `${snapshotId}_${counter}`）并建立 `idToNode` 映射，随后把结果写入 `#textSnapshot`。
  - 注意：为 option 节点补充 `value` 字段（当 option 的值被省略时，使用 name 作为 value）。
- `getTextSnapshot(): TextSnapshot | null`：读取缓存的 snapshot。
- `getAXNodeByUid(uid: string)`：从 `#textSnapshot.idToNode` 中按 uid 查找节点（可能返回 undefined）。
- `async getElementByUid(uid: string): Promise<ElementHandle<Element>>`
  - 描述：根据 snapshot 中的 uid 找到对应的 `TextSnapshotNode`，然后调用 `node.elementHandle()` 获取元素的真实 `ElementHandle`。此方法包含多层检查：
    - 若没有 snapshot，抛出提示使用 `takeSnapshot`；
    - 若 uid 的 snapshotId 与缓存 snapshot 不同，抛出“快照过期”错误；
    - 若找不到节点或 element handle，抛出“没有该元素”错误。
  - 语义：工具层在对 DOM 做动作（click/fill 等）前应尽可能先刷新 snapshot 并使用该方法以保证稳定引用。

###### 超时与节流（Timeouts / Throttling）
- `setNetworkConditions(conditions: string | null): void`
  - 将 `conditions` 写入 `#networkConditionsMap`（按 page），并调用 `#updateSelectedPageTimeouts()`。
- `getNetworkConditions(): string | null`
- `setCpuThrottlingRate(rate: number): void`
  - 将 CPU 倍率写入 `#cpuThrottlingRateMap` 并触发 `#updateSelectedPageTimeouts()`。
- `getCpuThrottlingRate(): number`
- `#updateSelectedPageTimeouts()`
  - 读取当前 page 的 cpu multiplier（默认为 1）和 network multiplier（由 `getNetworkMultiplierFromString` 计算），并调用：
    - `page.setDefaultTimeout(DEFAULT_TIMEOUT * cpuMultiplier)`
    - `page.setDefaultNavigationTimeout(NAVIGATION_TIMEOUT * networkMultiplier)`
  - 结果：所有后续的 `waitFor` / navigation API 会基于这些超时策略工作。工具层不需要自己重复计算超时，只要通过 context 设置节流/网络档位即可。

###### 等待策略支持（WaitForHelper）
- `getWaitForHelper(page, cpuMultiplier, networkMultiplier)`：返回一个新的 `WaitForHelper` 实例，用于在动作后等待网络/空闲事件，封装等待策略。
- `waitForEventsAfterAction(action: () => Promise<unknown>): Promise<void>`
  - 描述：工具层可以把一个异步动作（如 `() => page.goto(url)`）传入此方法，Context 内部会创建相关的 `WaitForHelper`（带当前 cpu/network 倍数）并执行 wait-for-logic，以保证在动作完成并达成稳定状态后再继续处理（例如在导航后等待网络空闲）。

###### 截图 / 临时文件保存
- `async saveTemporaryFile(data: Uint8Array<ArrayBufferLike>, mimeType: 'image/png' | 'image/jpeg' | 'image/webp'): Promise<{filename: string}>`
  - 描述：在系统临时目录下创建一个以 `chrome-devtools-mcp-` 前缀的临时目录，生成 filename（例如 `screenshot.png`），写入文件并返回 `{filename}`。若失败，记录日志并抛出带 cause 的错误。
- `async saveFile(data, filename): Promise<{ filename }>`
  - 描述：把数据写入指定绝对/相对路径的文件，返回 `{ filename }` 或在失败时抛错。

###### trace 相关
- `storeTraceRecording(result: TraceResult): void`：把解析后的 trace 结果压入 `#traceResults` 用于后续分析/查询。
- `recordedTraces(): TraceResult[]`：返回已记录的 trace 列表。
- `setIsRunningPerformanceTrace(x: boolean)` / `isRunningPerformanceTrace()`：标记是否正在进行 trace 录制（供工具或 UI 查询使用）。

###### collector / id 映射
- `getNetworkRequestStableId(request: HTTPRequest): number`：委托给 `#networkCollector`，得到某个请求的稳定 id，便于在 `McpResponse` 中引用请求。

##### 常见调用链（示例）
下面是工具层如何使用 `McpContext` 的典型示例片段（说明性伪代码）：

- 新建 page
```ts
// tools/pages.ts 的 handler
const page = await context.newPage(); // newPage 会自动注册 collectors 并更新 pages snapshot
response.setIncludePages(true);
```

- 关闭 page
```ts
try {
  await context.closePage(pageIdx);
  response.setIncludePages(true);
} catch (err) {
  if (err.message === CLOSE_PAGE_ERROR) {
    // 转换为用户可读错误
    return { error: '不能关闭最后一个页面' };
  }
  throw err;
}
```

- 导航并等待稳定
```ts
await context.waitForEventsAfterAction(() => page.goto(url));
// 或者工具先执行 action，再调用 wait helper 包装
```

- 使用快照定位元素
```ts
await context.createTextSnapshot();
const handle = await context.getElementByUid('5_12'); // 若 uid 过期或 snapshot 缺失会抛错
await handle.click();
```


##### 调试与测试建议（给维护者）
- 给 `McpContext` 写单元测试/集成测试要点：
  - 初始化后 `createPagesSnapshot()` 能正确读取 pages 列表。
  - `newPage()` 创建并注册 collectors（可以使用 mock page 触发网络/console 事件，断言 collector 收到）。
  - `createTextSnapshot()` 生成的 `idToNode` 中的 `id` 规则（如 `snapshotId_counter`）是稳定且唯一的。
  - `getElementByUid()` 在 snapshot 存在但元素 DOM 被移除时的报错路径。
  - `setCpuThrottlingRate` / `setNetworkConditions` 会修改 `page` 的默认 timeout（可能需要模拟 `Page`，或在 e2e 中断言 `page.getDefaultTimeout()` 的变化）。
- 为关键错误添加可机器识别的 error code（若需要被调用方 programmatically 区分），例如 `ERR_NO_SNAPSHOT` / `ERR_STALE_SNAPSHOT` / `ERR_LAST_PAGE_CLOSE`，这样工具层可以更精确地构造用户信息。

## 1.3 PageCollector — 每页事件收集器（简明说明）
文件：`src/PageCollector.ts`
### 概要
`PageCollector<T>` 是一个按 Page 分组收集浏览器事件/资源的通用组件。它为每个 Page 保持一个可变数组（按引用存储）来记录事件项，并给每个项分配一个稳定的 id（`stableIdSymbol`）。`NetworkCollector` 继承自它，重写了导航后清理策略以保留从最近一次导航起的请求。  
它封装了浏览器事件、资源的收集和处理过程。上下文可以调用其中的工具方法来获取页面数据或发起浏览器事件，Collector会对其统一进行处理、管理。（类似于为上下文实例提供了一个utils和管理机制）  




### 关键字段与行为


- id 生成器：从 1 开始自增，达到 Number.MAX_SAFE_INTEGER 时回绕到 0。
- `init()`：对已有 pages 进行初始化，并监听 browser 的 `targetcreated`/`targetdestroyed` 来增删 page 的监听器与存储。
- `addPage(page)`：显式为新 page 初始化收集（供工具在 `newPage()` 后调用）。上下文的newPage()会调用此方法。
- `#initializePage(page)`：设置 storage 数组、创建 listeners（由构造时传入的 initializer 产生），把 listeners 注册到 page，并在 main frame 导航时触发 `cleanupAfterNavigation`。
- `cleanupAfterNavigation(page)`（基类）：默认把该 page 的存储数组清空（保持引用）。
- `getData(page)`：读取该 page 的收集数组（可能为空数组）。
- `getById(page, stableId)`：按 stable id 查找并返回指定项（找不到抛错）。
- `getIdForResource(resource)`：读取资源上的 stable id（不存在返回 -1）。

### NetworkCollector 的特别之处（核心差异）
- 覆盖 `cleanupAfterNavigation(page)`：不是完全清空，而是找到最后一次针对 main frame 的 navigation 请求索引，并保留从那次请求开始的所有请求（含该 navigation 请求本身）。这让网络请求列表保持“最新导航相关”的上下文，便于分析单次导航期间的请求。

### 使用示例（伪代码，具体调用详见McpContext）
```ts
const netCollector = new NetworkCollector(browser, collect => ({
  request: (req) => collect(req)
}));
await netCollector.init();
const requests = netCollector.getData(selectedPage);
const stableId = netCollector.getIdForResource(requests[0]);
```

## 1.4 Pages
文件：`src/tools/pages.ts`
### 概要
pages.ts实现的是一组关于浏览器管理的对外的MCP工具命令，调用McpContext的低级方法，使用defineTool()定义了一不分直接可以由LLM调用的工具。  
定义的工具包括：listPages, selectPage, closePage, newPage, navigatePage, navigatePageHistory, resizePage, handleDialog，共8个，都是网页相关的（不包括模拟、性能测试等工具）。
这些工具与我们在basic_commands章节中介绍的工具一致。  

### 工具的定义
以selectPage为例，工具的定义方式如下：  
```ts
export const selectPage = defineTool({
  name: 'select_page',
  description: `Select a page as a context for future tool calls.`,
  annotations: {
    category: ToolCategories.NAVIGATION_AUTOMATION,
    readOnlyHint: true,
  },
  schema: {
    pageIdx: z
      .number()
      .describe(
        'The index of the page to select. Call list_pages to list pages.',
      ),
  },
  handler: async (request, response, context) => {
    const page = context.getPageByIdx(request.params.pageIdx);
    await page.bringToFront();
    context.setSelectedPageIdx(request.params.pageIdx);
    response.setIncludePages(true);
  },
});

```
我们定义了工具的名称、描述、参数、格式、处理函数。其中handler处理函数所调用的就是McpContext中的方法。  
如果你返回到Cherry Studio查看工具的描述，会发现与这里的description字段一致。  

### 工具的调用
tests/tools/pages.test.ts中提供了一个调用示例：  
```ts
describe('browser_select_page', () => {
    it('selects a page', async () => {
      await withBrowser(async (response, context) => {
        await context.newPage();
        assert.strictEqual(context.getSelectedPageIdx(), 1);
        await selectPage.handler({params: {pageIdx: 0}}, response, context);
        assert.strictEqual(context.getSelectedPageIdx(), 0);
        assert.ok(response.includePages);
      });
    });
  });

```
使用工具时，传入参数、返回、上下文，调用handler方法处理即可。  

### 小结
pages.ts 提供面向用户的页面控制工具：它做参数校验、调用 McpContext 完成实际工作、捕获并转换错误、并通过 McpResponse 指定需要返回的内容。McpContext 则是运行时的状态管理器和低级 Puppeteer 封装——工具层是“指挥者”，Context 是“执行者与状态库”。

## 1.5 本节小结
以上是Chrome DevTools MCP中的标签页与上下文管理具体实现模块。我们介绍了上下文状态的保存、上下文与Puppeteer低级工具的交互、状态管理的封装和MCP工具封装层。接下来，我们将介绍下一个模块：快照机制。  

