# `create_from_mermaid` 分析

## 1. 结论概览

`create_from_mermaid` 不是在 MCP Server 进程里直接把 Mermaid 文本转换成 Excalidraw 元素的工具，而是一个“前端代理转换”工具。

完整链路是：

`MCP tool call` -> `src/index.ts` -> `POST /api/elements/from-mermaid` -> `WebSocket broadcast` -> `浏览器前端执行 Mermaid 转换` -> `updateScene` -> `POST /api/elements/sync`

这意味着它有几个核心特征：

- 依赖已打开的 Excalidraw 前端页面
- 实际转换发生在浏览器里，不在 Node.js MCP 进程里
- MCP 返回成功，不等于 Mermaid 已经转换完成
- 当前实现更接近“用 Mermaid 结果替换当前画布”，而不是“向当前画布追加元素”

## 2. 实现位置

核心实现分布在以下文件中：

- MCP 工具注册：[src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:576)
- MCP 工具调用分发：[src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1337)
- Express 路由中转：[src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:692)
- 前端 WebSocket 消费与场景更新：[frontend/src/App.tsx](/var/www/github/mcp_excalidraw/frontend/src/App.tsx:683)
- Mermaid 转换封装：[frontend/src/utils/mermaidConverter.ts](/var/www/github/mcp_excalidraw/frontend/src/utils/mermaidConverter.ts:1)
- README 工具列表说明：[README.md](/var/www/github/mcp_excalidraw/README.md:433)

## 3. 输入与接口定义

`create_from_mermaid` 的 MCP schema 定义在 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:576)。

必填参数：

- `mermaidDiagram: string`

可选参数：

- `config.startOnLoad?: boolean`
- `config.flowchart?.curve?: 'linear' | 'basis'`
- `config.themeVariables?.fontSize?: string`
- `config.maxEdges?: number`
- `config.maxTextSize?: number`

示例输入：

```ts
{
  mermaidDiagram: "graph TD; A-->B; B-->C;",
  config: {
    flowchart: { curve: "linear" },
    themeVariables: { fontSize: "20px" }
  }
}
```

这些字段并不会在 MCP 层被真正消费，它们只是原样传给前端转换器。

## 4. 实际执行流程

### 4.1 MCP 层只负责转发

在 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1337) 中，工具调用逻辑先用 `zod` 校验参数，然后向 Express 服务发起请求：

```ts
const response = await fetch(`${EXPRESS_SERVER_URL}/api/elements/from-mermaid`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    mermaidDiagram: params.mermaidDiagram,
    config: params.config
  })
});
```

这里没有调用 `@excalidraw/mermaid-to-excalidraw`，也没有直接生成 Excalidraw 元素。MCP 层只是把 Mermaid 文本交给 HTTP 服务。

### 4.2 Express 路由负责广播请求

在 [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:692) 中，`POST /api/elements/from-mermaid` 会：

1. 校验 `mermaidDiagram` 是否存在且为字符串
2. 记录日志
3. 通过 WebSocket 广播一条 `mermaid_convert` 消息给所有前端客户端
4. 立即返回 success

广播的消息格式大致是：

```ts
{
  type: 'mermaid_convert',
  mermaidDiagram,
  config: config || {},
  timestamp: new Date().toISOString()
}
```

这个路由本身并不等待前端处理完成。

### 4.3 前端在浏览器中完成真正转换

前端在 [frontend/src/App.tsx](/var/www/github/mcp_excalidraw/frontend/src/App.tsx:683) 收到 `mermaid_convert` 后，会调用：

```ts
const result = await convertMermaidToExcalidraw(
  data.mermaidDiagram,
  data.config || DEFAULT_MERMAID_CONFIG
)
```

而 `convertMermaidToExcalidraw` 在 [frontend/src/utils/mermaidConverter.ts](/var/www/github/mcp_excalidraw/frontend/src/utils/mermaidConverter.ts:15) 中，底层使用的是：

```ts
parseMermaidToExcalidraw(mermaidDefinition, config)
```

注释也明确写了这一步依赖浏览器 DOM，必须运行在浏览器上下文中。

### 4.4 转换后直接更新当前场景

前端成功拿到结果后，会执行：

```ts
const convertedElements = convertToExcalidrawElements(result.elements, { regenerateIds: false })
applySceneUpdateWithoutAutoSync(excalidrawAPI, {
  elements: convertedElements,
  captureUpdate: CaptureUpdateAction.IMMEDIATELY
})
```

这里最重要的行为是：它直接把 `elements` 设置为 Mermaid 转换结果，没有把当前画布元素与之合并。

### 4.5 前端再把结果同步回服务端

在场景更新后，前端会调用：

```ts
await syncToBackend()
```

而 `syncToBackend()` 会把当前画布上的所有元素发给 `/api/elements/sync`。服务端在 [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:735) 中会先：

```ts
elements.clear()
```

然后再把前端传回来的元素重新写入内存存储。

因此 Mermaid 转换不仅更新了前端画布，也会把服务端的 canonical scene 一并替换掉。

## 5. 当前行为特征

### 5.1 这是异步中转，不是同步转换

MCP 返回值只说明“请求已经发给前端”，不说明：

- Mermaid 是否解析成功
- 前端是否真的在线
- 元素是否已经落到画布
- 服务端内存是否已经完成同步

这意味着在 `create_from_mermaid` 后立刻调用 `query_elements`、`describe_scene` 或 `export_scene`，理论上存在读到旧状态的窗口。

### 5.2 当前语义更像“替换画布”

虽然工具名叫 `create_from_mermaid`，但目前前端用的是：

```ts
updateScene({ elements: convertedElements })
```

而不是把 Mermaid 结果 append 到现有 scene。随后 `/api/elements/sync` 还会清空服务端内存并写入新场景。

所以它的真实语义更接近：

- “用 Mermaid 结果重建当前画布”
- 而不是“在当前画布中新增一组 Mermaid 元素”

### 5.3 依赖前端客户端存在

这个工具只有在浏览器前端已打开并完成 WebSocket 连接时才真正有意义。

如果没有前端客户端，Express 路由仍然会返回 success，但广播不会被任何人处理，最终不会生成元素。

## 6. 默认配置

前端默认 Mermaid 配置定义在 [frontend/src/utils/mermaidConverter.ts](/var/www/github/mcp_excalidraw/frontend/src/utils/mermaidConverter.ts:39)：

```ts
{
  startOnLoad: false,
  flowchart: {
    curve: 'linear',
  },
  themeVariables: {
    fontSize: '20px',
  },
  maxEdges: 500,
  maxTextSize: 50000,
}
```

如果 WebSocket 消息里没有提供 `config`，前端就会采用这份默认值。

## 7. 优点

### 7.1 复用官方/现有浏览器能力

项目没有在 Node 侧重写 Mermaid 布局或解析逻辑，而是复用 `@excalidraw/mermaid-to-excalidraw`。这降低了服务端实现复杂度，也避免了自己维护 Mermaid 到 Excalidraw 的转换器。

### 7.2 与现有实时画布架构兼容

当前项目本来就是“Express + WebSocket + 浏览器前端”结构。把 Mermaid 转换放在前端，可以自然接入已有的 `updateScene` 和 `syncToBackend` 流程。

### 7.3 适合快速导入结构化图

对于标准 Mermaid 定义，例如 flowchart、基础依赖关系图、ER 图等，这种方案适合快速生成一版初始图，然后再用 element-level 工具微调。

## 8. 问题与风险

### 8.1 没有前端客户端时会出现“假成功”

`POST /api/elements/from-mermaid` 没有像 `export_to_image` 那样检查 `clients.size === 0`。因此当前可能出现：

- MCP 返回 success
- 但没有任何浏览器处理 WebSocket 消息
- 最终画布和服务端状态都没有变化

这会让调用方误以为操作已经完成。

### 8.2 MCP 返回不包含最终结果

当前 MCP 返回的是“请求已发送”，而不是：

- 转换后的 element IDs
- 元素数量
- 转换耗时
- 失败原因
- 是否已同步回后端

这使得它不适合作为严格可编排、可验证的自动化步骤。

### 8.3 转换失败只能在浏览器控制台看到

前端出错时只会：

```ts
console.error('Mermaid conversion error:', result.error)
```

或：

```ts
console.error('Error converting Mermaid diagram from WebSocket:', error)
```

错误不会回传给 Express，也不会回传给 MCP 调用方。

### 8.4 会覆盖现有画布内容

当前实现不是 merge，而是 replace。对调用方来说这是一个高风险行为，因为工具名称没有明确传达“覆盖语义”。

如果用户希望：

- 保留现有图形
- 在某个区域追加 Mermaid 子图
- 只导入一部分 Mermaid 结构

当前实现都不满足。

### 8.5 多客户端连接可能产生竞态

服务端会把 `mermaid_convert` 广播给所有 WebSocket 客户端。多个打开的前端页面都会尝试：

1. 各自执行 Mermaid 转换
2. 各自 `updateScene`
3. 各自调用 `syncToBackend()`

这样会有几个问题：

- 重复计算
- 多次回写服务端
- 后写入者覆盖先写入者
- 如果不同浏览器环境产生非完全一致的结果，最终状态可能不稳定

### 8.6 结果存在时序窗口

因为调用链是：

`MCP -> Express -> WebSocket -> Frontend -> Sync`

所以任何依赖“转换结果已经存在”的后续操作，都有异步时序风险。当前没有 request/response ack 机制来收敛这个问题。

### 8.7 二次转换步骤可能带来字段偏差

`parseMermaidToExcalidraw()` 返回的已经是 Excalidraw elements，但前端又执行了一次：

```ts
convertToExcalidrawElements(result.elements, { regenerateIds: false })
```

这未必一定是错误，但它意味着 Mermaid converter 的输出还会再经过一次 Excalidraw 规范化流程。对于绑定、文本、分组或某些扩展字段，理论上存在被改写或丢失的可能性。

## 9. 与其他工具的关系

`create_from_mermaid` 和其他工具的定位不同：

- 相比 `create_element` / `batch_create_elements`：它不是精确控制单个元素，而是快速导入整张 Mermaid 图
- 相比 `import_scene`：它不是导入 `.excalidraw` JSON，而是导入 Mermaid DSL
- 相比 `describe_scene` / `get_canvas_screenshot`：它负责生成图，不负责理解图

一个更合理的使用方式通常是：

1. 用 `create_from_mermaid` 生成初稿
2. 等前端完成同步
3. 再用 `describe_scene`、`query_elements`、`update_element` 等工具细修

## 10. 改进建议

### 10.1 增加前端在线检查

在 `/api/elements/from-mermaid` 中增加：

```ts
if (clients.size === 0) {
  return res.status(503).json({
    success: false,
    error: 'No frontend client connected. Open the canvas in a browser first.'
  })
}
```

这可以避免“请求成功但没有任何实际效果”的假成功问题。

### 10.2 增加 request/response 确认机制

当前最值得做的改造是参考 `export_to_image` 的 pending request 机制：

- 服务端生成 `requestId`
- WebSocket 广播时带上 `requestId`
- 前端转换完成后回调一个结果接口
- 服务端等待 success / error / timeout
- MCP 返回真实转换结果

这样 MCP 调用就能获得：

- 是否成功
- 元素数量
- 失败原因
- 是否已完成同步

### 10.3 明确工具语义

如果产品设计就是“替换画布”，建议：

- 修改 tool description
- 在 README 和 skill 文档里明确说明会覆盖当前场景

如果设计目标是“追加”，则前端应改为：

- 读取当前 scene
- 合并 Mermaid elements
- 对新图做偏移或布局避让
- 再同步到后端

### 10.4 为多客户端选择单一执行者

当前所有客户端都会处理 `mermaid_convert`。更稳妥的做法是：

- 只让一个 active canvas client 执行转换
- 或让客户端通过 leader / session 标识决定谁处理

这样可以避免重复同步和竞态覆盖。

### 10.5 返回结构化结果

即使短期内不做完整 ack 机制，也可以考虑让前端回传：

- `count`
- `elementIds`
- `filesCount`
- `syncCompleted`

便于后续自动化流程串联。

## 11. 适用场景与不适用场景

适用：

- 已有 Mermaid 图，希望快速导入到 Excalidraw
- 需要先生成结构草图，再人工或 Agent 继续微调
- 浏览器前端已打开，允许异步完成转换

不适用：

- 纯后端无浏览器环境
- 需要立即拿到精确元素结果
- 需要保证不覆盖当前画布
- 需要严格可重复、可等待、可断言的自动化流水线

## 12. 总结

`create_from_mermaid` 的核心价值是“借助浏览器前端，把 Mermaid DSL 快速转成 Excalidraw 初稿”。它实现简单，和当前项目架构兼容，也适合快速导入标准化结构图。

但从工程语义上看，它目前仍是一个弱确认、强依赖前端、带覆盖行为的异步工具。最大的问题不是转换能力本身，而是调用成功与实际完成之间存在明显鸿沟。

如果后续希望把它作为稳定的 Agent 自动化能力使用，最重要的改造方向不是优化 Mermaid 解析，而是补齐“在线检查、完成确认、错误回传、结果结构化”这条控制链路。
