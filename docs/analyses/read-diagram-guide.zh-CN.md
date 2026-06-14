# `read_diagram_guide` 分析

## 1. 结论概览

`read_diagram_guide` 不是读取当前画布或分析已有图形的工具，而是一个面向 AI Agent 的静态设计规范工具。它的作用是把一份 Excalidraw 图形设计指南注入到 LLM 上下文中，帮助 Agent 在创建图形前统一颜色、尺寸、间距、箭头绑定和常见反模式。

当前实现非常轻量：工具无输入参数、无外部依赖、无副作用，调用时直接返回一段硬编码 Markdown 文本。

## 2. 实现位置

核心实现集中在 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:279)。

相关位置：

- 设计指南常量：`DIAGRAM_DESIGN_GUIDE`，定义于 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:279)
- MCP 工具注册：`read_diagram_guide`，定义于 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:796)
- 工具调用分发：`case 'read_diagram_guide'`，定义于 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1911)
- README 能力说明：见 [README.md](/var/www/github/mcp_excalidraw/README.md:78)
- Agent skill 工作流引用：见 [skills/excalidraw-skill/SKILL.md](/var/www/github/mcp_excalidraw/skills/excalidraw-skill/SKILL.md:136)

## 3. 当前行为

### 3.1 输入

该工具没有输入参数：

```ts
inputSchema: {
  type: 'object',
  properties: {}
}
```

因此它不能根据图类型、用户目标、当前画布状态或输出风格动态调整内容。

### 3.2 输出

调用后返回 MCP text content：

```ts
return {
  content: [{ type: 'text', text: DIAGRAM_DESIGN_GUIDE }]
};
```

返回内容是一段 Markdown 文本，主要包括：

- 颜色规范：stroke colors 和 fill colors
- 尺寸规范：最小形状尺寸、字号、内边距、箭头长度
- 布局规范：网格对齐、元素间距、流向、层级和分组
- 箭头规范：推荐使用 `startElementId` / `endElementId` 绑定箭头
- 图类型模板：Architecture Diagram、Flowchart、ER Diagram
- 反模式：重叠、拥挤、小字号、手写箭头坐标、颜色过多等
- 推荐绘制顺序：背景区域、主体形状、箭头、注释、视觉检查

### 3.3 副作用

该工具没有副作用：

- 不读取 canvas
- 不访问前端
- 不调用 Canvas Server HTTP API
- 不修改元素
- 不依赖浏览器是否打开
- 不依赖当前画布是否为空

它更像是一个“运行时可发现的内置文档接口”。

## 4. 与现有工具体系的关系

`read_diagram_guide` 的内容和元素创建 API 是配套的。

例如 guide 中建议使用的字段，在 `create_element` / `batch_create_elements` schema 中都有对应能力：

- `backgroundColor`
- `strokeColor`
- `strokeStyle`
- `text`
- `fontSize`
- `startElementId`
- `endElementId`
- `startArrowhead`
- `endArrowhead`

这说明该工具不是孤立文档，而是在指导 Agent 使用项目已经封装好的高层字段。尤其是 `startElementId` / `endElementId`，它和当前项目对箭头自动绑定、自动路由的封装方向一致。

在 Agent workflow 中，skill 文档也明确建议：创建新图前先调用 `read_diagram_guide`，再规划坐标网格、批量创建元素、绑定箭头、设置视口、截图检查。

## 5. 优点

### 5.1 简单可靠

实现只是返回常量字符串，没有网络、文件、浏览器或画布状态依赖，因此失败面很小。

这类工具适合作为 LLM 调用前置步骤，因为它不会污染画布，也不会引入状态变化。

### 5.2 降低 Agent 出图随机性

AI 生成图形时常见问题包括元素太小、间距不足、颜色杂乱、箭头不绑定、标签缺失。guide 把这些问题直接转成规则，可以明显约束 Agent 的默认行为。

### 5.3 和项目定位一致

本项目强调的不是一次性生成图片，而是可迭代、可检查、可编辑的画布工作流。`read_diagram_guide` 正好补充了“怎么画得更规范”的指导层。

### 5.4 对 MCP 客户端友好

MCP 客户端可以通过 `tools/list` 发现该工具，并在需要时主动调用。相比只写在 README 中，作为 tool 暴露更容易进入模型上下文。

## 6. 问题与风险

### 6.1 内容硬编码在 `src/index.ts`

当前 `DIAGRAM_DESIGN_GUIDE` 是一大段 Markdown 字符串，直接放在 MCP server 主入口文件中。

这带来几个问题：

- 增加 `src/index.ts` 文件体积
- 业务逻辑和文档内容耦合
- 修改文案需要改代码文件
- 不利于复用到 README、skill 或外部文档
- 后续容易出现多处文档漂移

更合理的方式是把它移动到独立 Markdown、JSON 或 TypeScript module 中。

### 6.2 返回值缺少结构化信息

当前只返回 Markdown 文本。对于 LLM 阅读足够，但对于程序化消费不够友好。

例如客户端无法直接读取：

- 可用颜色列表
- 推荐字号范围
- 图类型模板
- 反模式列表
- 每类 diagram 的推荐尺寸

如果后续希望让其他工具或前端 UI 消费这份 guide，纯 Markdown 会成为限制。

### 6.3 不支持按场景裁剪

工具没有参数，因此不能针对具体目标返回更聚焦的指南。

例如这些场景目前都会得到同一份内容：

- 架构图
- 流程图
- ER 图
- 时序图
- 产品流程图
- 云资源拓扑
- 当前画布修复建议

对 Agent 来说，完整指南有帮助，但也会增加上下文噪音。

### 6.4 不读取当前画布，名称可能造成误解

`read_diagram_guide` 的名字是准确的，因为它读的是 guide。但如果用户只看到 `read_diagram` 这一部分，可能误以为它能读取或分析当前 diagram。

当前真正用于理解画布的是：

- `describe_scene`
- `get_canvas_screenshot`
- `get_element`
- `query_elements`

因此文档中最好持续强调：`read_diagram_guide` 是设计指南，不是场景分析工具。

### 6.5 与 skill 文档存在轻微表述冲突

skill 文档提醒：如果给背景区域 rectangle 直接设置 `text` / `label.text`，标签会居中显示，可能覆盖区域内的其他元素；更好的做法是使用独立 text 元素放在 zone 顶部。

但 guide 中 Architecture Diagram 段落写的是：

```md
Zones: large light-gray background rectangles with 20px fontSize labels
```

这句话没有明确说明 zone label 应该使用独立 text 元素，可能让 Agent 误解为可以直接给背景矩形设置 `text`。建议补充说明：背景 zone 使用浅色矩形，标题使用独立 text 元素放在区域顶部。

### 6.6 缺少独立测试

当前没有看到专门覆盖该工具的测试。虽然实现很简单，但仍建议增加轻量测试，防止以下问题：

- 工具未注册
- 工具名被误改
- input schema 变化
- handler 返回空文本
- 文档关键段落被误删

## 7. 改进建议

### 7.1 抽离 guide 内容

建议把 `DIAGRAM_DESIGN_GUIDE` 从 `src/index.ts` 中抽离。

可选方案：

- `src/guides/diagram-design-guide.ts`：继续导出字符串，改动最小
- `docs/guides/diagram-design-guide.md`：文档和运行时共用，但需要构建或运行时读取
- `src/guides/diagram-design-guide.json`：适合结构化消费，但阅读性略差

短期推荐第一种：新建 `src/guides/diagram-design-guide.ts`，导出常量，`src/index.ts` 只负责 import。

### 7.2 增加结构化版本

可以保持当前 Markdown 输出兼容，同时增加结构化字段。

例如：

```ts
{
  content: [
    { type: 'text', text: DIAGRAM_DESIGN_GUIDE }
  ],
  metadata: {
    palettes: [...],
    sizingRules: {...},
    antiPatterns: [...]
  }
}
```

如果 MCP SDK 不适合直接放 metadata，也可以新增工具，例如：

- `read_diagram_guide`
- `read_diagram_guide_json`

但除非有明确消费方，否则不建议过早增加工具数量。

### 7.3 增加可选参数

可以给工具增加可选参数：

```ts
{
  diagramType?: 'architecture' | 'flowchart' | 'er' | 'generic',
  format?: 'markdown' | 'json'
}
```

这样既保持兼容，又能减少上下文噪音。

### 7.4 修正文案中的 zone label 指引

建议把 Architecture Diagram 中的 zone 描述改为类似：

```md
- Zones: use large light-gray background rectangles with low opacity; add zone titles as separate text elements near the top-left, not bound rectangle text.
```

这样和 skill 文档中的质量检查规则保持一致。

### 7.5 增加最小测试

建议至少覆盖：

- `tools/list` 包含 `read_diagram_guide`
- input schema 为无参数 object
- 调用后返回 text content
- 返回内容包含关键标题，例如 `Color Palette`、`Sizing Rules`、`Anti-Patterns`

如果项目后续引入快照测试，也可以对 guide 输出做 snapshot，但要注意避免文案小改导致测试过脆。

## 8. 推荐优先级

建议按以下顺序处理：

1. 修正 zone label 文案，避免 Agent 画出覆盖内容的背景区域标签。
2. 把 guide 常量抽离到独立文件，降低 `src/index.ts` 维护负担。
3. 增加轻量测试，保证工具注册和返回内容稳定。
4. 如果后续确实有前端或外部消费需求，再考虑结构化返回。
5. 如果 Agent 上下文成本成为问题，再增加 `diagramType` 参数做按场景裁剪。

## 9. 总体评价

`read_diagram_guide` 是一个低复杂度、高收益的 Agent 辅助工具。它不解决画布状态问题，也不负责图形生成，但能显著改善 Agent 在使用 `create_element` / `batch_create_elements` 时的默认审美和工程约束。

当前最大的问题不是功能缺失，而是维护形态偏粗糙：内容硬编码、缺少测试、和 skill 文档存在轻微表述不一致。只要把 guide 抽离、修正文案并补上基础测试，这个工具就能成为比较稳定的“设计规范入口”。
