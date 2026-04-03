# ClawX 学习文档

## 1. 这个项目是什么

ClawX 是一个基于 **Electron + React + TypeScript + Vite** 的跨平台桌面应用。

它不是一个“直接实现大模型能力”的项目，而是一个给 **OpenClaw** 运行时做图形界面的桌面客户端。你可以把它理解成：

- OpenClaw 负责“AI 代理运行、工具调用、消息通道、调度、技能执行”
- ClawX 负责“桌面 UI、系统集成、配置管理、把 OpenClaw 能力以图形界面展示出来”

一句话概括：

**ClawX = OpenClaw 的桌面控制台。**

---

## 2. 这个项目能做什么

从 README、代码结构和页面设计来看，它主要提供这些能力：

- 聊天界面：和 OpenClaw 代理对话，支持流式消息、工具调用展示、会话切换
- Agents 页面：管理代理
- Channels 页面：配置消息渠道和多账号
- Skills 页面：管理技能
- Cron 页面：定时任务和自动化
- Models 页面：查看 token usage、模型使用情况、成本统计
- Settings 页面：配置主题、语言、代理商、网关、更新、开发者选项
- Setup 首次引导：第一次启动时引导用户完成运行时检查和 provider 配置

---

## 3. 技术栈总览

### 前端

- React 19
- React Router 7
- Zustand
- Tailwind CSS
- Radix UI
- Framer Motion
- i18next

### 桌面层

- Electron 40
- preload + contextBridge
- IPC

### 构建与工程化

- Vite
- vite-plugin-electron
- TypeScript
- ESLint
- pnpm

### 测试

- Vitest + jsdom
- Testing Library
- Playwright（Electron E2E）

### 后端/运行时接入

- OpenClaw
- Host API（Main 进程里的本地 HTTP server）
- WebSocket / HTTP / IPC 多传输策略

---

## 4. 项目目录怎么读

建议你先记住下面几个目录：

```text
electron/
  main/        Electron 主进程
  preload/     安全桥接层
  api/         Host API，本地 HTTP 服务和路由
  gateway/     OpenClaw Gateway 的进程管理和连接管理
  services/    主进程服务，如 provider 管理
  utils/       主进程工具函数

src/
  pages/       页面
  components/  组件
  stores/      Zustand 状态管理
  lib/         前端 API 封装、事件订阅、工具函数
  i18n/        多语言
  types/       类型定义

shared/
  目前较少，放跨端复用逻辑

tests/
  unit/        单元测试
  e2e/         Electron 端到端测试

scripts/
  打包、下载 OpenClaw、comms 回放、生成资源等脚本

resources/
  图标、截图、预装技能、CLI 资源
```

如果你是新手，推荐按这个顺序读代码：

1. `package.json`
2. `vite.config.ts`
3. `electron/main/index.ts`
4. `electron/preload/index.ts`
5. `src/main.tsx`
6. `src/App.tsx`
7. `src/pages/Chat/index.tsx`
8. `src/stores/gateway.ts`
9. `src/stores/chat.ts`
10. `electron/api/server.ts`
11. `electron/gateway/manager.ts`

---

## 5. 应用启动时到底发生了什么

### 5.1 启动入口

开发时执行：

```bash
pnpm dev
```

Vite 会同时做两件事：

- 启动 React renderer 开发服务
- 编译并启动 Electron 主进程和 preload

`vite.config.ts` 里用了 `vite-plugin-electron`，分别指定了两个入口：

- `electron/main/index.ts`
- `electron/preload/index.ts`

而前端入口是：

- `src/main.tsx`

### 5.2 Electron 主进程启动

主进程入口是 `electron/main/index.ts`。

它负责：

- 保证单实例运行
- 初始化日志、代理设置、遥测、开机启动
- 创建主窗口和托盘
- 注册 IPC handlers
- 启动 Host API server
- 创建并管理 `GatewayManager`
- 自动启动 OpenClaw Gateway
- 安装内置 skills / 预装 skills / bundled plugins

也就是说，**主进程是整个桌面应用的“总控中心”**。

### 5.3 preload 负责安全桥接

`electron/preload/index.ts` 通过 `contextBridge.exposeInMainWorld('electron', ...)` 给渲染进程暴露有限能力。

这里做了两件关键事：

- 白名单限制哪些 IPC channel 可以调用
- 不把 Node 能力直接暴露给 React 页面

这符合 Electron 的安全模型：

- renderer 不直接拿 Node API
- renderer 只能通过 preload 暴露的受控接口和 main 通信

### 5.4 React 渲染层启动

`src/main.tsx` 做的事很少：

- 初始化 i18n
- 初始化 API transport
- 用 `HashRouter` 挂载 `App`

`src/App.tsx` 才是前端总入口，它做这些初始化：

- 初始化 settings store
- 初始化 gateway store
- 初始化 provider store
- 同步主题和语言
- 如果没完成 setup，跳转到 `/setup`
- 注册来自 main 的导航事件

---

## 6. 整体架构怎么理解

这是这个项目最重要的一部分。

### 6.1 架构分层

可以把它分成 4 层：

```text
React 页面 / Zustand Store
        ↓
src/lib/host-api.ts + src/lib/api-client.ts
        ↓
Electron Main（IPC + Host API + GatewayManager）
        ↓
OpenClaw Gateway（独立运行时进程）
```

### 6.2 角色划分

#### Renderer

也就是 `src/` 里的 React 代码。

职责：

- 负责 UI 展示
- 负责用户交互
- 调用统一 API 封装
- 订阅事件更新状态

不应该做的事：

- 不直接访问系统资源
- 不直接调用本地 Gateway HTTP 地址
- 不直接新增裸 `window.electron.ipcRenderer.invoke(...)`

项目规范已经明确：

- Renderer 应通过 `src/lib/host-api.ts` 和 `src/lib/api-client.ts`
- 不要在页面/组件里直接去打 `http://127.0.0.1:18789`

#### Preload

职责：

- 暴露受控 API 给 renderer
- 做 IPC channel 白名单

#### Main

职责：

- 管理 Electron 生命周期
- 管理窗口、托盘、更新、系统设置
- 维护 Host API 和 IPC handlers
- 管理 OpenClaw Gateway 进程

#### Gateway

职责：

- 运行 OpenClaw
- 执行 agent/chat/rpc/channel/cron/skill 等能力

---

## 7. 通信链路怎么走

这个项目的通信不是单一方式，而是 **IPC + Host API + 事件流** 混合。

### 7.1 请求链路

典型请求链路：

```text
页面/Store
  -> hostApiFetch() 或 invokeIpc()
  -> main 进程
  -> Host API route 或 IPC handler
  -> GatewayManager
  -> OpenClaw Gateway
```

### 7.2 为什么既有 IPC 又有 Host API

因为这个项目在逐步统一接口。

现在有两套主要模式：

#### 模式 A：Host API

前端调用 `src/lib/host-api.ts` 的 `hostApiFetch('/api/...')`

流程：

1. renderer 调用 `hostApiFetch`
2. 默认不是直接 fetch localhost
3. 而是先通过 IPC 调用 `hostapi:fetch`
4. main 代理请求到本地 Host API server
5. route handler 处理逻辑

优点：

- 接口统一，像普通前后端分层
- 避免 renderer 直接面向 main 的细碎 IPC
- 避免 dev/prod 环境下 CORS 和地址差异

#### 模式 B：统一 IPC / legacy IPC

前端调用 `invokeIpc('provider:list')` 或 `invokeApi(...)`

`src/lib/api-client.ts` 做了 transport 策略：

- 默认优先 IPC
- 某些网关 RPC 可支持 WS / HTTP / IPC fallback
- 当前默认策略仍然是“IPC 为主，WS/HTTP 作为诊断或回退能力”

### 7.3 事件链路

事件主要有两种来源：

- main 通过 IPC 主动推送给 renderer
- Host API 的 SSE 作为补充

`src/lib/host-events.ts` 会优先走 IPC 事件映射，例如：

- `gateway:status`
- `gateway:error`
- `gateway:notification`
- `gateway:chat-message`

只有当本地允许 SSE fallback 时才会走 `EventSource`。

后端事件源在：

- `electron/api/event-bus.ts`

这是一个简单的 SSE 客户端集合管理器。

---

## 8. OpenClaw Gateway 是怎么被管理的

核心类是：

- `electron/gateway/manager.ts`

这是整个项目最重要的后端类之一。

它负责：

- 启动 Gateway 进程
- 停止 Gateway 进程
- 重启 Gateway
- 建立 WebSocket 连接
- 做心跳和健康检查
- 处理断线重连
- 处理进程 ownership
- 转发运行时事件给 UI

### 8.1 为什么这里复杂

因为桌面应用和本地运行时之间经常会出现这些问题：

- 端口被占用
- 进程崩溃
- WebSocket 断开
- 主程序退出但子进程没退出
- Windows 上重复实例和端口竞争

所以这个模块做了大量“工程性保护”：

- single instance lock
- file lock
- reconnect/backoff
- restart governor
- heartbeat timeout
- emergency cleanup
- doctor repair

如果你未来要调“为什么 UI 一直显示 connecting”“为什么 Gateway 重启了”“为什么消息没回来”，优先看这个目录：

```text
electron/gateway/
```

---

## 9. Host API 是什么

主进程里起了一个本地 HTTP 服务：

- `electron/api/server.ts`

它不是给公网访问的，而是给本机 app 自己使用。

特性：

- 监听 `127.0.0.1`
- 启动时生成随机 token
- 每个请求必须带 Bearer token
- 对 mutation 请求要求 `application/json`
- 统一分发到各个 route handler

路由文件在：

```text
electron/api/routes/
```

例如：

- `gateway.ts`
- `settings.ts`
- `providers.ts`
- `agents.ts`
- `channels.ts`
- `cron.ts`
- `skills.ts`
- `usage.ts`

这套设计很像一个小型后端服务，只是它运行在 Electron main 里。

### 9.1 一个具体例子

`electron/api/routes/gateway.ts` 里定义了：

- `GET /api/gateway/status`
- `GET /api/gateway/health`
- `POST /api/gateway/start`
- `POST /api/gateway/stop`
- `POST /api/gateway/restart`

所以当前端要查网关状态时，不是直接写系统逻辑，而是走：

```text
useGatewayStore
  -> hostApiFetch('/api/gateway/status')
  -> handleGatewayRoutes()
  -> gatewayManager.getStatus()
```

这就是这个项目推荐的新增功能方式。

---

## 10. Zustand 状态管理怎么组织

核心 store 在：

```text
src/stores/
```

常见的有：

- `settings.ts`
- `gateway.ts`
- `providers.ts`
- `agents.ts`
- `channels.ts`
- `skills.ts`
- `cron.ts`
- `chat.ts`
- `update.ts`

### 10.1 settings store

`src/stores/settings.ts`

特点：

- 使用 Zustand + persist
- renderer 里保留一份本地持久化缓存
- 启动时再从 Host API 拉主进程中的真实设置
- 修改设置时，通常先更新前端状态，再异步同步到 Host API

这是一个很常见的桌面应用设计：

- UI 立刻响应
- 后台再持久化

### 10.2 gateway store

`src/stores/gateway.ts`

职责：

- 初始化网关状态
- 订阅网关事件
- 转发事件给 chat store / channels store
- 暴露 start/stop/restart/rpc

这里你能看到一个很典型的模式：

- 页面不直接自己处理网关事件
- store 层集中处理，再驱动 UI

### 10.3 chat store

`src/stores/chat.ts`

这是前端最复杂的 store。

它负责：

- 会话列表
- 当前会话
- 消息历史
- 发送消息
- 流式消息
- thinking/tool 状态
- 附件缓存
- 历史轮询
- 去重
- 错误恢复

你可以把它理解成“聊天领域的状态机”。

如果你是新手，不要一开始就想把这个文件全看懂。建议拆成几个问题读：

1. 消息从哪里发出去
2. 流式返回怎么拼接
3. 什么时候刷新历史
4. 什么时候切换 session
5. 出错时 UI 怎么恢复

---

## 11. 页面层怎么组织

页面在：

```text
src/pages/
```

主要页面：

- `Chat/`
- `Models/`
- `Agents/`
- `Channels/`
- `Skills/`
- `Cron/`
- `Settings/`
- `Setup/`

### 11.1 App 路由

`src/App.tsx` 里路由很清楚：

- `/setup/*`
- `/`
- `/models`
- `/agents`
- `/channels`
- `/skills`
- `/cron`
- `/settings/*`

### 11.2 MainLayout

`src/components/layout/MainLayout.tsx`

页面骨架非常简单：

- 顶部 `TitleBar`
- 左侧 `Sidebar`
- 中间 `Outlet`

这意味着大多数新功能都应该被看成：

- 放进某个 page
- 复用 layout
- 通过 store 和 API 获取数据

### 11.3 Setup 页面

`src/pages/Setup/index.tsx`

首次启动流程包含：

- Welcome
- Runtime 检查
- Provider 配置
- Skills 安装
- Complete

这就是“第一次使用”的 onboarding。

### 11.4 Chat 页面

`src/pages/Chat/index.tsx`

页面结构是：

- `ChatToolbar`
- 消息列表
- 流式消息占位
- error bar
- `ChatInput`

页面本身不做重业务，它主要从 chat store 取状态，然后渲染。

这也是这个项目一个重要风格：

- 复杂逻辑尽量进 store / lib
- page 负责组合和展示

---

## 12. Provider、Settings、Secrets 是怎么管理的

Provider 相关逻辑主要在：

```text
electron/services/providers/
electron/shared/providers/
src/lib/providers.ts
src/components/settings/ProvidersSettings.tsx
src/stores/providers.ts
```

大致分工：

- renderer：展示 provider 配置 UI
- main service：保存、校验、同步 provider
- secrets：API key 放系统安全存储或安全层，而不是纯前端 localStorage

你可以把它理解成：

- 配置对象存在 app store / 主进程配置层
- 敏感凭证有单独的 secret 存储通道
- 修改 provider 后，还要同步到 OpenClaw runtime

所以 provider 不是简单“存个 JSON”。

---

## 13. Models 页面里的 token usage 从哪来

很多新手会以为这种统计是从控制台日志里算出来的，其实不是。

项目约定非常明确：

- 它读取 OpenClaw session transcript 的 `.jsonl` 文件
- 扫描 agent 的 session 目录
- 解析里面结构化 usage 字段
- 聚合成 token/cost/model 统计

核心代码在：

- `electron/utils/token-usage.ts`
- `electron/utils/token-usage-core.ts`
- `src/pages/Models/index.tsx`

这说明一个重要设计思想：

**UI 的统计不是“猜测日志”，而是读结构化历史记录。**

这是很值得学习的工程实践。

---

## 14. 测试体系怎么读

### 14.1 单元测试

配置在：

- `vitest.config.ts`
- `tests/setup.ts`

特点：

- 运行环境是 `jsdom`
- 已经 mock 了 `window.electron`
- 单元测试主要覆盖 store、页面、工具函数、gateway 逻辑

你写前端逻辑或纯函数时，优先补单元测试。

### 14.2 E2E 测试

配置在：

- `playwright.config.ts`
- `tests/e2e/fixtures/electron.ts`

这里不是测网页，而是直接拉起 Electron。

fixture 做了这些事：

- 用临时目录模拟 HOME 和 userData
- 动态分配 Host API 端口
- 启动编译后的 `dist-electron/main/index.js`
- 提供稳定窗口获取逻辑
- 支持跳过 setup

项目规范还要求：

**任何用户可见的 UI 变更，都应该带上或更新一个 Electron E2E spec。**

这是你开发时必须记住的规则。

---

## 15. 构建和打包怎么工作

### 15.1 开发构建

开发态：

```bash
pnpm dev
```

会启动：

- Vite renderer
- Electron main/preload 构建
- Electron app

### 15.2 生产打包

主要命令：

```bash
pnpm run build
pnpm run package:win
pnpm run package:mac
pnpm run package:linux
```

### 15.3 electron-builder

配置在：

- `electron-builder.yml`

重点：

- `dist` 和 `dist-electron` 被打进应用
- `resources/` 会作为 extraResources 带进去
- OpenClaw、预装技能、CLI、插件也会一起带上
- 按平台分别配置图标、安装器和资源目录

这说明 ClawX 不是单纯前端壳子，它是一个 **自带运行时资源的桌面分发包**。

---

## 16. 你要开发新功能时，应该从哪层入手

这是最实用的一节。

### 场景 A：只是改页面样式

去这些位置：

- `src/pages/...`
- `src/components/...`
- `src/styles/globals.css`

一般不需要碰 main。

### 场景 B：页面要读取新的设置或数据

推荐路径：

1. main 里加 Host API route 或统一 IPC 支持
2. renderer 的 `src/lib/host-api.ts` / `src/lib/api-client.ts` 调用它
3. 在 `src/stores/...` 里封装状态
4. 页面消费 store

### 场景 C：要接 OpenClaw 新能力

推荐路径：

1. 看 OpenClaw Gateway 暴露什么 RPC / 配置
2. 在 `GatewayManager` 或 route handler 里接进去
3. 在 store 里处理状态与事件
4. 页面展示

### 场景 D：要新增系统能力

比如：

- 打开文件
- 打开路径
- 调系统对话框
- 开机启动
- 系统托盘

这种一般属于：

- `electron/main/`
- `electron/utils/`
- `electron/preload/`

不是 renderer 直接做。

---

## 17. 这个项目最值得学习的工程点

### 17.1 边界清晰

它没有把所有逻辑都塞进 React 组件，而是分成：

- page
- component
- store
- lib
- main
- gateway

这很适合中大型应用。

### 17.2 桌面安全模型比较规范

- contextIsolation 开启
- preload 白名单
- renderer 不直接拿 Node
- 外链协议做了限制

### 17.3 通信层设计有演进意识

虽然现在 IPC 和 Host API 并存，但你能看出它在往“统一接口层”演进。

### 17.4 对运行时故障有很多工程保护

尤其是 `electron/gateway/`，这部分很适合学习“桌面 app 怎么管理本地后台服务”。

### 17.5 测试覆盖面不错

不仅有单元测试，还有 Electron E2E，这对桌面项目很重要。

---

## 18. 新手最容易懵的点

### 18.1 为什么 renderer 不能直接 fetch 127.0.0.1:18789

因为项目明确要求由 main 持有传输策略，避免：

- CORS 问题
- dev/prod 行为不一致
- renderer 里出现环境漂移逻辑

### 18.2 为什么既有 Host API 又有 IPC

因为项目处在演进过程中：

- 老能力还在走 IPC
- 新能力更倾向 Host API / 统一请求协议

### 18.3 为什么 GatewayManager 这么复杂

因为本地进程管理天然很复杂，不是“spawn 一个子进程”就结束。

### 18.4 为什么 chat store 很大

因为聊天产品本身就是状态机：

- 流式输出
- tool 调用
- session 切换
- 失败恢复
- 附件
- 去重

这些都是真实复杂度，不是代码写得“故意大”。

---

## 19. 推荐你的学习路线

### 第一阶段：先能跑起来

1. 安装依赖
2. 执行 `pnpm run init`
3. 执行 `pnpm dev`
4. 看应用有哪些页面
5. 跑一次 `pnpm test`

### 第二阶段：理解启动链路

按顺序读：

1. `package.json`
2. `vite.config.ts`
3. `electron/main/index.ts`
4. `electron/preload/index.ts`
5. `src/main.tsx`
6. `src/App.tsx`

目标：

- 知道谁先启动
- 知道窗口何时创建
- 知道 setup 为什么会先出现

### 第三阶段：理解通信链路

按顺序读：

1. `src/lib/api-client.ts`
2. `src/lib/host-api.ts`
3. `src/lib/host-events.ts`
4. `electron/api/server.ts`
5. `electron/api/routes/gateway.ts`
6. `electron/main/ipc-handlers.ts`

目标：

- 知道请求怎么从 UI 到 main
- 知道事件怎么从 main 回到 UI

### 第四阶段：理解聊天主流程

按顺序读：

1. `src/pages/Chat/index.tsx`
2. `src/pages/Chat/ChatInput.tsx`
3. `src/pages/Chat/ChatMessage.tsx`
4. `src/stores/gateway.ts`
5. `src/stores/chat.ts`

目标：

- 知道发消息时发生什么
- 知道为什么会有 streaming/pendingFinal/tool 状态

### 第五阶段：理解运行时管理

按顺序读：

1. `electron/gateway/manager.ts`
2. `electron/gateway/process-launcher.ts`
3. `electron/gateway/ws-client.ts`
4. `electron/gateway/startup-orchestrator.ts`
5. `electron/gateway/supervisor.ts`

目标：

- 知道 OpenClaw 是怎么被启动和监管的

### 第六阶段：开始做小改动

适合新手的练手任务：

1. 给 Settings 页新增一个只影响前端显示的小开关
2. 给某个列表加搜索或筛选
3. 给某个页面补一个 loading/error 态
4. 给 Chat 页加一个纯 UI 的辅助展示
5. 给某个工具函数补单元测试

不要一上来改 `gateway/manager.ts` 或大改 `chat.ts`。

---

## 20. 开发时的实际建议

### 20.1 先找“入口文件”

比如你想改 Channels 页面，不要一开始全局搜索。先看：

- `src/pages/Channels/index.tsx`
- 它用了哪些 store
- store 调了哪些 API

### 20.2 遇到复杂 store，先看 action，不要先看 state

比如 chat store，优先搜：

- `sendMessage`
- `loadHistory`
- `loadSessions`
- `handleChatEvent`

### 20.3 遇到 main 逻辑，先画链路

比如“保存 provider”：

```text
页面点击保存
-> store action
-> invokeIpc / hostApiFetch
-> main handler / route
-> provider service
-> runtime sync
```

先画出来，再读代码。

### 20.4 每次只追一个问题

不要一次追完整个系统。比如今天只追：

- 为什么 setup 完成后会跳首页

或者只追：

- 为什么 Gateway 状态能实时刷新

这样效率最高。

---

## 21. 你以后改代码时要遵守的项目规则

这些规则非常重要：

- Renderer 侧统一通过 `src/lib/host-api.ts` 和 `src/lib/api-client.ts`
- 不要在组件里新增裸 `window.electron.ipcRenderer.invoke(...)`
- 不要在 renderer 里直接访问 Gateway HTTP 端点
- 涉及 UI 可见变化，最好补 E2E
- 功能或架构改动后，要检查 README 多语言文档是否需要同步
- 如果改了通信链路，推送前应跑：

```bash
pnpm run comms:replay
pnpm run comms:compare
```

---

## 22. 常用命令

```bash
pnpm run init
pnpm dev
pnpm run lint
pnpm run typecheck
pnpm test
pnpm run test:e2e
pnpm run build:vite
pnpm run comms:replay
pnpm run comms:compare
```

---

## 23. 最后给新手的结论

如果你把这个项目看成“一个 React 应用”，你会很快迷路。

正确理解方式是：

- 它首先是一个 **Electron 桌面应用**
- 其次它带有一个 **本地后端层（main + Host API）**
- 最后它连接一个 **独立的 OpenClaw 运行时**

所以你开发它时，要始终问自己：

1. 这个功能属于 renderer、main，还是 gateway？
2. 这个请求应该走 Host API 还是统一 IPC？
3. 这是 UI 状态，还是运行时状态？
4. 这个变化要不要补单元测试或 E2E？

只要这四个问题想清楚，这个项目就不会乱。

---

## 24. 下一步我建议你怎么学

如果你现在就想开始真正上手，我建议下一步做这三件事：

1. 我带你按顺序精读 `electron/main/index.ts`
2. 我带你追一遍“Chat 页面发消息”的完整调用链
3. 我给你设计一个适合新手的第一个小功能，并带你改完

如果你愿意，下一条我可以直接开始做第 1 步：**逐行带你读 `electron/main/index.ts`。**
