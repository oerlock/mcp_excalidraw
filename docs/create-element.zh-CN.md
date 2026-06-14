# `create_element` 逻辑说明

本文整理 `create_element` 的实际行为，覆盖 MCP 工具层、HTTP 画布服务、前端渲染链路，以及元素类型与 `x` / `y` 计算规则。

相关实现：

- MCP 工具入口：`src/index.ts`
- HTTP 画布服务：`src/server.ts`
- 前端画布渲染：`frontend/src/App.tsx`

## 1. `create_element` 是什么

`create_element` 不是直接在浏览器画布上调用某个“绘图函数”。

它的职责更准确地说是：

1. 在 MCP 层接收更适合 Agent 调用的参数
2. 把参数整理成 Excalidraw / Canvas Server 可接受的数据结构
3. 通过 HTTP 请求发送给 Canvas Server
4. 由 Canvas Server 更新内存状态并广播
5. 由前端收到广播后，把元素真正渲染到 Excalidraw 画布上

因此，它本质上是一个“协议适配 + 数据转发”的工具入口，而不是最终渲染器。

## 2. 端到端调用链路

典型链路如下：

1. MCP 客户端调用 `create_element`
2. `src/index.ts` 中的 MCP handler 校验参数并组装元素对象
3. MCP Server 调用 `POST /api/elements`
4. `src/server.ts` 校验并存储元素
5. Canvas Server 通过 WebSocket 广播 `element_created`
6. `frontend/src/App.tsx` 收到消息后调用 Excalidraw API 更新场景
7. Excalidraw React 组件完成最终渲染

从职责边界上看：

- MCP 层负责输入适配
- HTTP 服务负责存储、广播、箭头几何修正
- 前端负责把服务端元素转成 Excalidraw 原生结构并显示在画布上

## 3. MCP 层的 `create_element` 做了什么

### 3.1 工具定义

MCP 工具定义在 `src/index.ts` 的 `tools` 数组中，对外暴露的工具名是 `create_element`。

必填参数：

- `type`
- `x`
- `y`

常见可选参数：

- `width`
- `height`
- `backgroundColor`
- `strokeColor`
- `strokeWidth`
- `strokeStyle`
- `roughness`
- `opacity`
- `text`
- `fontSize`
- `fontFamily`
- `startElementId`
- `endElementId`
- `startArrowhead`
- `endArrowhead`

### 3.2 参数校验

真正用于运行时校验的是 `ElementSchema`，而不是仅靠工具对外 schema 文本描述。

`ElementSchema` 除了上面这些字段，还支持：

- `points`
- `groupIds`
- `locked`
- `roundness`
- `fillStyle`
- `elbowed`

这意味着：

- MCP 工具“元数据里写出来的字段”不是完整能力集合
- 真正能否传入，最终由 `ElementSchema` 决定

### 3.3 元素组装逻辑

MCP handler 会做这几件事：

1. 读取 `id`
   - 如果调用方传了 `id`，则直接使用
   - 否则调用 `generateId()` 生成
2. 展开基础属性
   - `x`、`y`、`width`、`height`、样式字段等直接进入元素对象
3. 规范化 `points`
   - 如果 `points` 使用 `{x, y}` 结构，会转成 `[x, y]`
4. 处理箭头绑定
   - `startElementId` 转成 `start: { id: ... }`
   - `endElementId` 转成 `end: { id: ... }`
5. 归一化字体
   - `fontFamily` 字符串名会转成 Excalidraw 期待的数值枚举
6. 补齐元数据
   - `createdAt`
   - `updatedAt`
   - `version: 1`

### 3.4 `text` 到 `label` 的转换

MCP 层有一层很重要的适配：

- 如果元素类型是 `text`，则保留 `text`
- 如果元素类型是 `rectangle`、`ellipse`、`diamond` 等 shape，且传了 `text`，则改写成 `label: { text }`

这样做的目的，是让调用方不用直接理解 Excalidraw 的容器文本结构。

### 3.5 MCP 层不负责真正绘制

MCP 层最后会调用 HTTP 画布服务，而不会在本地保存场景：

- `createElementOnCanvas(...)`
- `POST /api/elements`

所以它做的是“把创建请求交给画布服务”。

## 4. HTTP 画布服务的创建逻辑

HTTP 创建逻辑在 `src/server.ts` 的 `POST /api/elements`。

### 4.1 请求体校验

服务端先用 `CreateElementSchema` 校验请求体。

它支持的字段比 MCP 对外 schema 更完整，除了常见图形字段外，还支持：

- `label`
- `points`
- `start`
- `end`
- `startBinding`
- `endBinding`
- `boundElements`
- `fileId`
- `status`
- `scale`

这说明：

- HTTP 层比 MCP 层更接近 Excalidraw 底层结构
- 同一个元素，通过 REST 直调时可操作的字段范围更大

### 4.2 组装标准化元素对象

服务端会拼出这样的对象：

```ts
const element: ServerElement = {
  id,
  ...params,
  fontFamily: normalizeFontFamily(params.fontFamily),
  createdAt: new Date().toISOString(),
  updatedAt: new Date().toISOString(),
  version: 1
};
```

这里的逻辑是：

- `id` 优先使用调用方传入值
- `...params` 保留所有合法输入字段
- `fontFamily` 做统一归一化
- 时间戳和 `version` 由服务端补齐

### 4.3 箭头和线的绑定解析

如果元素类型是：

- `arrow`
- `line`

服务端会调用 `resolveArrowBindings([element])`。

它会：

1. 读取 `start.id` / `end.id`
2. 查找对应目标元素
3. 计算目标元素中心点
4. 根据目标方向，求图形边缘交点
5. 加一个固定 `8px` 的 gap
6. 重写箭头的 `x`、`y`、`points`

也就是说：

- 普通图形的 `x` / `y` 基本保持原值
- 绑定箭头的 `x` / `y` 可能被服务端重新计算

### 4.4 存储与广播

元素对象组装完成后，服务端会：

1. 写入内存 `elements` Map
2. 通过 WebSocket 广播 `element_created`
3. 返回 JSON：

```json
{
  "success": true,
  "element": { "...": "..." }
}
```

注意：这里是内存存储，不是持久化存储。服务重启后，元素会丢失。

## 5. 真正把元素画到画布上的逻辑在哪

真正渲染发生在前端，而不是 `create_element` 或 `POST /api/elements` 中。

前端链路在 `frontend/src/App.tsx`：

1. WebSocket 收到 `element_created`
2. 调用 `cleanElementForExcalidraw(...)` 去掉服务端元数据
3. 调用 `convertElementsPreservingImageProps(...)`
4. 内部调用 `convertToExcalidrawElements(...)`
5. 最终调用 `excalidrawAPI.updateScene(...)`
6. `<Excalidraw />` 组件完成屏幕渲染

所以从工程上看：

- 后端负责“状态”
- 前端负责“显示”
- Excalidraw 第三方组件负责“实际绘制像素”

## 6. `create_element` 可以创建哪些类型

类型枚举来自 `EXCALIDRAW_ELEMENT_TYPES`。

当前支持：

- `rectangle`
- `ellipse`
- `diamond`
- `arrow`
- `text`
- `freedraw`
- `line`
- `image`

### 6.1 完整支持度说明

虽然 `image` 在类型枚举里存在，但 MCP 的 `create_element` 对外 schema 没有完整暴露图片所需字段，例如：

- `fileId`
- `status`
- `scale`

因此从使用体验上看：

- `rectangle`、`ellipse`、`diamond`、`arrow`、`text`、`line` 支持最完整
- `freedraw` 次之
- `image` 类型在 MCP 工具层属于受限支持，REST 直调更完整

## 7. `x` / `y` 是怎么计算的

### 7.1 普通元素

对于这些普通元素：

- `rectangle`
- `ellipse`
- `diamond`
- `text`
- `image`
- `freedraw`

`x` 和 `y` 基本不做计算，直接使用调用方传入值。

也就是说：

- `x` 是元素左上角横坐标
- `y` 是元素左上角纵坐标

当前服务端不会帮你自动布局、居中或避让。

### 7.2 未绑定的线和箭头

如果是 `arrow` 或 `line`，但没有绑定其他元素：

- `x` / `y` 仍然以调用方传值为准
- `points` 决定线段形状

这种情况下，服务端不会根据其他元素重算几何位置。

### 7.3 绑定到其他元素的箭头

如果调用时传了：

- `startElementId`
- `endElementId`

那么最终流程是：

1. MCP 层把它们改写成 `start` / `end`
2. HTTP 服务根据目标元素位置重算箭头
3. 最终写回新的：
   - `x`
   - `y`
   - `points`

因此绑定箭头的 `x` / `y` 只是“初始值”，最终结果以服务端重算为准。

### 7.4 不同图形边缘点的计算

服务端 `computeEdgePoint(...)` 对不同图形做了不同处理：

- `diamond`
  - 按菱形几何求边缘交点
- `ellipse`
  - 按椭圆参数方程求边缘交点
- 其他图形
  - 默认按矩形边界计算交点

这保证箭头连接点更接近图形外边缘，而不是简单连接中心点。

## 8. 成功语义上的一个注意点

MCP 层存在一个需要注意的实现细节：

- `syncToCanvas(...)` 失败时会返回 `null`
- `createElementOnCanvas(...)` 会退回原始 `elementData`
- `create_element` handler 可能仍然返回“创建成功”

这意味着：

- MCP 调用返回成功，不一定代表 HTTP 画布服务真的持久保存了元素
- 如果你遇到“工具说成功，但画布没显示”，要重点检查：
  - `EXPRESS_SERVER_URL`
  - Canvas Server 是否运行
  - `/api/elements` 是否真的返回 200

## 9. 适合怎么使用 `create_element`

`create_element` 适合：

- 创建单个 shape
- 创建单个文本
- 创建一条连接到已有元素的箭头
- 小步迭代式改图

不太适合：

- 一次创建多元素且相互依赖的复杂结构
- 图片元素的完整创建流程

对于“先建多个节点，再互相连线”的场景，`batch_create_elements` 通常更稳，因为批量接口更适合处理同批次元素之间的 ID 引用关系。

## 10. 延伸阅读

如果你关心“画布上已经有元素时，Agent 应该怎样自动推一个合理的追加位置”，单独见：

- [追加元素时如何自动推一个合理的 `x` / `y`](./append-element-positioning.zh-CN.md)

## 11. 总结

`create_element` 的本质可以概括为一句话：

它不是“直接画图”的函数，而是一个把 Agent 友好输入转换为 Excalidraw 画布服务输入，再通过前端完成真实渲染的创建入口。

最核心的价值在于两点：

- 对 shape 的 `text -> label` 转换
- 对箭头的 `startElementId` / `endElementId` 绑定适配

最需要注意的点也有两点：

- 绑定箭头的 `x` / `y` 会被服务端重算
- MCP 返回成功不一定等于画布服务已成功落图
