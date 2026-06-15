# `get_canvas_screenshot` 分析

## 结论

`get_canvas_screenshot` 是一个面向 MCP 调用方的“画布视觉校验”工具。它本身不直接截图，也不在服务端渲染 PNG，而是通过已连接的 Excalidraw 前端页面导出当前画布，再把结果以 `image/png` 的形式返回给调用方。

这意味着它更准确的描述是：

- 返回“当前场景导出的 PNG”
- 不是“浏览器窗口的像素级截图”
- 依赖前端页面在线并已连接到服务端

## 入口位置

工具定义位于：

- [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:783)

工具处理逻辑位于：

- [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1873)

定义上的关键信息：

- 工具名：`get_canvas_screenshot`
- 描述：获取当前画布截图并作为图片返回
- 参数：仅支持一个可选参数 `background`

## 调用链

整体调用链如下：

1. MCP 工具接收请求
2. MCP 向 Express 服务发起 `POST /api/export/image`
3. Express 通过 WebSocket 通知前端执行导出
4. 前端调用 Excalidraw 导出 API 生成 PNG
5. 前端将 base64 图片数据回传给 Express
6. MCP 将结果包装为 `image/png` 返回

### 1. MCP 工具层

`get_canvas_screenshot` 在 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1873) 中：

- 用 `zod` 解析参数，只接受 `background?: boolean`
- 记录日志 `Taking canvas screenshot via MCP`
- 固定请求 `/api/export/image`
- 固定传入 `format: 'png'`

实际请求体等价于：

```json
{
  "format": "png",
  "background": true
}
```

当请求成功后，它不会写文件，也不会返回纯文本，而是直接返回 MCP 的图片内容：

```ts
{
  type: 'image',
  data: result.data,
  mimeType: 'image/png'
}
```

因此，这个工具的定位非常明确：给支持图片内容的调用方直接消费。

### 2. Express 中转层

服务端入口在：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:859)

这一层做了几件关键事情。

#### 参数校验

服务端要求：

- `format` 必须存在
- `format` 只能是 `png` 或 `svg`

虽然 `get_canvas_screenshot` 只传 `png`，但这条接口是复用给 `export_to_image` 的，因此保留了通用能力。

#### 前端连接检查

如果没有任何前端客户端连接，服务端直接返回 `503`：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:870)

错误信息也很直接：

- `No frontend client connected. Open the canvas in a browser first.`

这也是 README 中明确写出的限制：

- [README.md](/var/www/github/mcp_excalidraw/README.md:495)

#### 注册一次待完成导出

服务端会生成一个 `requestId`，并把这次导出挂到 `pendingExports` 中等待结果：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:877)

这里的 `PendingExport` 结构包含：

- `resolve`
- `reject`
- `timeout`
- `collectionTimeout`
- `bestResult`

说明这不是一个同步导出接口，而是一个“请求 - 广播 - 收集结果 - 汇总返回”的异步编排。

#### 导出前先广播规范状态

服务端在真正请求导出前，会先广播一次：

- `initial_elements`

代码位置：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:894)

作用是让所有连接中的前端客户端先和服务端持有的 canonical scene 对齐，避免前端本地状态滞后导致导出旧图。

#### 固定等待 800ms 再请求导出

服务端随后使用 `setTimeout(..., 800)` 再广播：

- `export_image_request`

代码位置：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:904)

这是一个明显的工程折中：作者没有实现“前端确认同步完成后再导出”的握手机制，而是采用了经验值等待时间。

### 3. 前端实际导出

前端处理逻辑位于：

- [frontend/src/App.tsx](/var/www/github/mcp_excalidraw/frontend/src/App.tsx:534)

前端收到 `export_image_request` 后：

- 读取当前 `elements`
- 读取当前 `appState`
- 读取当前 `files`

如果格式是 PNG，则调用：

- `exportToBlob(...)`

然后通过 `FileReader.readAsDataURL(blob)` 读出 base64，再回传给：

- `POST /api/export/image/result`

这意味着最终图像并不是对浏览器页面做 DOM/Canvas 截屏，而是由 Excalidraw 基于当前场景重新导出的结果。

### 4. 回传与聚合

结果接收接口位于：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:938)

服务端处理结果时有两个重要策略。

#### 单个前端失败不会立刻整体失败

如果某个客户端回传了 `error`，服务端只记录 warning，不立即结束这次导出。这样做的假设是：

- 可能有多个前端客户端在线
- 某一个客户端失败，不代表其他客户端也失败

#### 保留“更大的结果”

服务端通过下面的逻辑保留最佳结果：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:961)

策略是：

- 如果新回传的数据更长，就替换掉当前 `bestResult`

注释写的是：

- `most complete canvas state wins`

这个判断方式是启发式的，不是强一致性判断，但实现成本低，且在多客户端结果不完全一致时能给出一个务实选择。

#### 结果收集窗口

第一个成功结果回来后，服务端不会立刻返回，而是再开启一个 3 秒窗口继续收集：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:966)

超出这个窗口后，服务端用当前 `bestResult` 作为最终结果。

此外还有一个总超时：

- 30 秒

代码位置：

- [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:879)

## 与 `export_to_image` 的关系

`get_canvas_screenshot` 与 `export_to_image` 共用同一条后端导出链路。

`export_to_image` 位于：

- [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1615)

二者差别主要在 MCP 返回形式。

### `get_canvas_screenshot`

- 固定输出 `png`
- 返回 MCP `image` 内容
- 用途是“让模型直接看当前图长什么样”

### `export_to_image`

- 支持 `png` / `svg`
- 可返回文本、base64，或落盘到文件
- 用途更偏“导出资产”或“保存文件”

所以可以把 `get_canvas_screenshot` 理解成 `export_to_image(format=png)` 的一个“多模态友好封装”。

## 这个工具实际返回的是什么

这个问题很关键。

它返回的是：

- 当前 Excalidraw 场景的 PNG 导出结果

它不是：

- 浏览器窗口截图
- 用户操作系统层面的屏幕截图
- 当前 viewport 可见区域的像素快照

这几个概念不要混淆。

从实现上看，前端是拿 `getSceneElements()`、`getAppState()`、`getFiles()` 交给 Excalidraw 的导出能力处理，因此它更像“场景渲染导出”，而不是“UI 表面截图”。

## 优点

### 1. 能形成视觉反馈闭环

`describe_scene` 提供结构化理解，`get_canvas_screenshot` 提供视觉验证。这使得 agent 可以：

- 先理解当前元素分布
- 再检查实际观感
- 再进行位置、大小、文本、连线的调整

这也是 README 中强调的 closed feedback loop。

### 2. 职责边界合理

服务端负责：

- 管理规范状态
- 协调导出流程
- 聚合多个前端结果

前端负责：

- 调用 Excalidraw 官方导出能力
- 生成真实的 PNG / SVG 数据

这比在 Node 侧强行模拟渲染环境更稳。

### 3. 对多客户端场景有一定容错

实现允许多个前端同时在线，并且：

- 某个客户端失败不会直接拖垮整体导出
- 可以收集多个结果再挑一个最优候选

虽然策略简单，但在实际协作场景里是有价值的。

## 风险与局限

### 1. 强依赖浏览器前端在线

这是最直接的运行前提。

如果前端页面没有打开，或者 WebSocket 没连上，工具必然失败。这一点不是边缘情况，而是设计前提。

### 2. `800ms` 等待是经验值，不是同步保证

服务端先广播 `initial_elements`，再等待 800ms 发出导出请求。这只能“提高同步完成的概率”，不能严格保证：

- 前端已经完成场景应用
- 字体/资源已经稳定
- 多客户端状态完全一致

如果前端负载较高、浏览器卡顿、网络抖动，仍然可能导出到旧状态或半稳定状态。

### 3. “更大的结果更完整”只是启发式判断

服务端使用 `data.length` 判断哪个结果更好。这个方法简单，但不严格。

可能存在的反例：

- 某个结果更大只是因为编码差异
- 某个前端渲染出的内容不一致但字节更多
- 较大的结果并不一定就是最新状态

当前实现中，这个策略够用，但不能被视为严格正确。

### 4. 返回的不是用户当前看到的 viewport 截图

如果调用方把这个工具理解为“把浏览器当前看到的区域截一张图”，就会产生预期偏差。

它更接近：

- 导出整个当前场景

而不是：

- 截图当前镜头位置下的画布视图

### 5. PNG 路径比 SVG 路径更脆弱

SVG 直接序列化字符串即可回传。

PNG 需要：

1. `exportToBlob`
2. `FileReader.readAsDataURL`
3. 再从 Data URL 中拆出 base64

因此 PNG 这条链路上的错误点更多，包括：

- Blob 生成失败
- FileReader 失败
- base64 提取失败

虽然代码已经做了错误上报，但路径本身确实更复杂。

## 适合的使用方式

`get_canvas_screenshot` 最适合以下场景：

- 在批量修改元素后检查整体观感
- 配合 `describe_scene` 做“结构 + 视觉”的双重验证
- 确认文本是否截断
- 确认节点是否重叠
- 确认箭头是否穿插混乱
- 确认布局是否符合预期

不太适合直接拿来做：

- 精准 viewport 截屏
- 无浏览器环境下的服务端静态导出
- 对渲染时机有严格确定性要求的流程

## 如果要继续改进，优先级最高的方向

如果后续要增强这个工具，实现上最值得优先考虑的是：

### 1. 增加“前端同步完成”确认机制

替代固定 800ms 等待，改为：

- 服务端广播 `initial_elements`
- 前端完成应用后显式 ack
- 服务端收到 ack 再触发导出

这样可以明显降低竞态风险。

### 2. 更明确地区分“场景导出”和“viewport 截图”

如果产品语义上真的需要“用户当前看到的视图截图”，应该单独提供新工具，而不是继续复用当前导出机制。

### 3. 为多客户端选择结果建立更强规则

比如：

- 只接受最新同步版本的客户端结果
- 只选择主客户端结果
- 在回传结果里带上 scene version 再判断

这会比单纯比较 base64 长度更稳。

## 总结

`get_canvas_screenshot` 的本质不是“截图功能”，而是“通过前端 Excalidraw 导出当前画布，并把 PNG 作为 MCP 图片返回”的桥接工具。

它的价值在于让 agent 具备真正的视觉校验能力，和 `describe_scene` 形成闭环；它的主要限制在于强依赖前端在线，以及当前导出链路仍然包含基于时间等待的同步假设。

如果只用一句话概括：

- `get_canvas_screenshot` 返回的是“当前画布场景的 PNG 导出结果”，不是“浏览器原样屏幕截图”。
