# MCP Excalidraw 项目说明

## 1. 项目定位

`mcp_excalidraw` 是一个面向 AI Agent 的 Excalidraw 画布编排项目，目标不是一次性“生成一张图”，而是提供一个可持续交互的实时画布，让 Agent 能像操作白板一样逐步创建、观察、修改和导出图形内容。

项目主要服务两类使用方式：

- 作为 **MCP Server** 接入 Claude Desktop、Cursor、Codex CLI 等 MCP 客户端
- 作为 **画布服务** 提供可视化前端、REST API 和 WebSocket 实时同步

它和官方偏“一次 prompt 生成图”的 Excalidraw MCP 的差异在于：本项目强调元素级操作、状态保留、可视反馈和迭代修图。

## 2. 核心能力

项目当前提供的能力可以分成四组：

### 2.1 元素级画布操作

- 创建、更新、删除单个元素
- 批量创建元素
- 查询元素、读取单个元素
- 清空画布
- 复制元素
- 锁定与解锁元素
- 分组与解组
- 对齐与分布

### 2.2 场景级操作

- 导出当前画布为 `.excalidraw` JSON
- 从 `.excalidraw` 文件或原始 JSON 导入
- 保存快照
- 恢复快照
- 输出结构化场景描述

### 2.3 浏览器协同能力

以下能力依赖前端画布已在浏览器中打开并连接 WebSocket：

- 画布截图
- PNG / SVG 导出
- 视口控制
- Mermaid 转换后的前端落图

### 2.4 Agent 辅助能力

- 内置 diagram guide，约束颜色、间距、字号、布局模式
- skill 文档支持 MCP 模式与 REST API 模式切换
- 支持导出为 shareable Excalidraw URL

## 3. 总体架构

项目由 4 个部分组成：

### 3.1 MCP Server

入口文件：[src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:1)

职责：

- 通过 `stdio` 暴露 MCP tools
- 接收来自 MCP 客户端的工具调用
- 将元素操作转发到 Canvas Server 的 HTTP API
- 封装更适合 LLM 使用的输入格式，例如：
  - 将 shape 的 `text` 自动转成 Excalidraw 需要的 `label`
  - 将 `startElementId` / `endElementId` 自动转成箭头绑定结构

特点：

- 当前没有独立持久化存储
- 更像“协议适配层 + 能力编排层”
- 工具定义集中在 `tools` 数组中，维护成本相对集中

### 3.2 Canvas Server

入口文件：[src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:1)

职责：

- 提供 REST API
- 承载 WebSocket 连接
- 维护服务端内存中的元素状态、文件状态和快照状态
- 对前端广播元素变更
- 提供健康检查、截图导出、视口控制等服务端接口

特点：

- 基于 Express + WebSocket
- 默认仅监听 `127.0.0.1`
- 没有内建鉴权

### 3.3 Frontend

入口文件：

- [frontend/src/main.tsx](/var/www/github/mcp_excalidraw/frontend/src/main.tsx:1)
- [frontend/src/App.tsx](/var/www/github/mcp_excalidraw/frontend/src/App.tsx:1)

职责：

- 渲染 Excalidraw 画布
- 接收 WebSocket 推送并更新画布
- 将服务端元素转换成 Excalidraw 可接受的结构
- 执行浏览器侧导出能力
- 处理 Mermaid 转换

前端构建通过 Vite 完成，配置见 [vite.config.js](/var/www/github/mcp_excalidraw/vite.config.js:1)。

### 3.4 Agent Skill

说明文件：[skills/excalidraw-skill/SKILL.md](/var/www/github/mcp_excalidraw/skills/excalidraw-skill/SKILL.md:1)

职责：

- 规范 Agent 在使用该项目时的工作流
- 优先走 MCP 模式，失败时降级到 REST API
- 提供布局、标注、箭头绑定、导图质量检查等操作规则

这部分不是运行时代码，但对 Agent 的实际使用效果影响很大。

## 4. 运行链路

项目运行时通常分成两个独立进程：

### 4.1 Canvas 进程

负责提供：

- Web UI
- REST API
- WebSocket 实时同步

默认地址：

- `http://127.0.0.1:3000`

### 4.2 MCP 进程

负责提供：

- MCP 工具调用接口

运行时通过环境变量 `EXPRESS_SERVER_URL` 指向 Canvas Server。

因此典型调用链路是：

1. MCP 客户端调用某个 tool
2. `src/index.ts` 解析参数并补齐 Excalidraw 所需结构
3. MCP Server 通过 HTTP 请求访问 Canvas Server
4. Canvas Server 更新内存状态
5. Canvas Server 通过 WebSocket 推送给浏览器前端
6. 前端画布实时刷新

## 5. 目录说明

```text
.
├── frontend/                 # React + Vite 前端
│   └── src/
├── scripts/                  # 辅助脚本
├── skills/
│   └── excalidraw-skill/     # Agent skill 说明与脚本
├── src/                      # TypeScript 后端代码
│   ├── index.ts              # MCP Server 入口
│   ├── server.ts             # Canvas Server 入口
│   ├── types.ts              # 共享类型、内存状态
│   └── utils/
├── Dockerfile                # MCP Server 镜像
├── Dockerfile.canvas         # Canvas Server 镜像
├── docker-compose.yml        # 本地组合运行
├── package.json
├── tsconfig.json
└── vite.config.js
```

## 6. 数据与状态模型

当前状态全部保存在内存中，定义见 [src/types.ts](/var/www/github/mcp_excalidraw/src/types.ts:286)：

- `elements`: 画布元素 `Map`
- `snapshots`: 快照 `Map`
- `files`: 图片类元素依赖的二进制文件 `Map`

这意味着：

- 服务重启后，状态会丢失
- 不适合作为长期持久化协作白板
- 更适合作为本地 Agent 工具链的一环

项目现在的导入导出能力，本质上承担了“临时持久化”的职责。

## 7. API 与工具面

### 7.1 MCP 工具

MCP 工具统一定义在 [src/index.ts](/var/www/github/mcp_excalidraw/src/index.ts:373)，大致包括：

- 元素 CRUD
- 布局工具
- Mermaid 转换
- 场景导入导出
- 快照
- 截图与图片导出
- 场景描述
- 视口控制

这层对 LLM 最友好，因为做了输入结构适配。

### 7.2 REST API

主要路由在 [src/server.ts](/var/www/github/mcp_excalidraw/src/server.ts:231) 一带，核心包括：

- `/api/elements`
- `/api/elements/:id`
- `/api/elements/batch`
- `/api/elements/from-mermaid`
- `/api/export/image`
- `/api/viewport`
- `/api/snapshots`
- `/health`

REST API 更适合作为调试接口和 MCP 的后端依赖。

### 7.3 WebSocket

WebSocket 用于：

- 新客户端接入时下发当前全量元素
- 元素创建、更新、删除广播
- 导出与视口控制请求回传

它是前端“实时同步画布”的关键。

## 8. 构建与启动方式

### 8.1 本地启动

典型流程：

```bash
npm ci
npm run build
PORT=3000 npm run canvas
```

然后在另一个终端启动 MCP：

```bash
EXPRESS_SERVER_URL=http://127.0.0.1:3000 node dist/index.js
```

### 8.2 Docker

仓库提供两套镜像：

- [Dockerfile](/var/www/github/mcp_excalidraw/Dockerfile:1)：仅 MCP Server
- [Dockerfile.canvas](/var/www/github/mcp_excalidraw/Dockerfile.canvas:1)：Canvas Server

组合启动见 [docker-compose.yml](/var/www/github/mcp_excalidraw/docker-compose.yml:1)。

设计意图很明确：

- MCP 是“协议接入层”
- Canvas 是“可视运行层”

两者可以分开部署。

## 9. 关键实现特点

### 9.1 MCP 层做了格式适配

项目对 Agent 很友好的一个点是，MCP 层帮忙抹平了 Excalidraw 的部分内部细节：

- shape 允许直接传 `text`
- 箭头允许直接传 `startElementId` / `endElementId`
- `fontFamily` 支持字符串名映射到数值

这显著降低了 Agent 直接使用 Excalidraw 数据结构的复杂度。

### 9.2 前端承担了部分浏览器专属能力

像截图、导图、viewport 控制，本质上无法完全在纯 Node 进程中完成，所以这里采用了服务端发请求、前端执行、再回传结果的方式。这种设计合理，但会引入对“浏览器端在线”的依赖。

### 9.3 文档与 Skill 的耦合度较高

这个项目的可用性不只来自代码，也来自 Skill 中对布局规则、反模式和质量检查流程的约束。换句话说，这个仓库的一部分产品能力体现在“指导 Agent 如何使用它”。

## 10. 当前限制与风险

### 10.1 无持久化

所有状态都在内存里，进程重启即丢失。

### 10.2 无鉴权

HTTP API 没有认证和权限控制，不适合直接暴露到不受控网络。

### 10.3 高级能力依赖前端在线

没有浏览器连接时，截图、图片导出、视口控制等能力不可用或不完整。

### 10.4 发布版本信息不一致

当前仓库存在版本口径不一致现象：

- `package.json` 中版本为 `1.0.7`
- `src/index.ts` 中 MCP server 元信息声明为 `2.0.0`
- `README` 以 `v2.0` 描述新能力

这会影响发布管理、排错和用户认知。

### 10.5 依赖可复现性一般

`package.json` 中 `@modelcontextprotocol/sdk` 使用了 `latest`，未来可能因上游变化导致构建或运行行为漂移。

## 11. 适用场景

这个项目适合：

- AI Agent 自动绘制架构图、流程图、关系图
- 需要迭代修图、观察结果、再继续修改的场景
- 本地开发环境中的可视化辅助工具
- 通过 MCP 为多个 AI 客户端统一提供 Excalidraw 操作能力

不太适合：

- 长期在线协作白板产品
- 强持久化、多租户、权限体系完整的生产级 SaaS 服务
- 完全无人值守且无浏览器参与的导图场景

## 12. 建议的后续演进方向

如果后续要把这个项目从“本地 Agent 工具”继续往“可稳定运行的服务”推进，优先级建议如下：

1. 增加持久化层，至少支持元素与快照落盘
2. 为 HTTP API 增加认证与最小权限控制
3. 统一版本号与发布流程
4. 锁定关键依赖版本，提升可复现性
5. 为前端在线依赖增加更明确的错误提示与状态反馈
6. 补充自动化测试，覆盖 MCP 到 Canvas 的关键链路

## 13. 总结

`mcp_excalidraw` 的核心价值不在“生成一张 Excalidraw 图”，而在“给 AI 一个可编排、可观察、可迭代修改的实时画布”。这使它非常适合作为 Agent 的图形执行层。

从工程形态上看，它当前更像一个本地优先的可视化工具链组件，而不是一个完整的持久化在线产品。只要明确这一定位，它的设计基本是自洽的：MCP 负责协议与抽象，Canvas 负责状态与同步，前端负责渲染与浏览器能力，Skill 负责把这些能力真正变成 Agent 可稳定使用的工作流。
