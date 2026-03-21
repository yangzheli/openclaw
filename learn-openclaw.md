# OpenClaw 代码结构与原理分析

> 基于 openclaw 源码的深度分析，记录核心架构设计和工作原理。

---

## 目录

1. [项目结构概览](#1-项目结构概览)
2. [核心架构原理](#2-核心架构原理)
3. [消息处理流程](#3-消息处理流程)
4. [飞书 Channel 实现](#4-飞书-channel-实现)
5. [Agents 人设存储机制](#5-agents-人设存储机制)
6. [关键设计模式](#6-关键设计模式)

---

## 1. 项目结构概览

### 1.1 目录结构

```
openclaw/
├── src/                    # 核心 TypeScript 源码
│   ├── cli/               # CLI 命令行接口
│   ├── commands/          # 各命令实现
│   ├── gateway/           # Gateway 服务器核心
│   ├── channels/          # 消息通道插件系统
│   ├── routing/           # 消息路由逻辑
│   ├── agents/            # AI Agent 运行时
│   ├── plugins/           # 插件系统核心
│   ├── plugin-sdk/        # 插件 SDK 公共 API
│   ├── acp/               # Agent Control Protocol
│   ├── config/            # 配置管理
│   └── infra/             # 基础设施/工具
├── extensions/            # 扩展插件包
│   ├── feishu/           # 飞书 Channel 插件
│   ├── msteams/          # Microsoft Teams 插件
│   ├── matrix/           # Matrix 插件
│   └── ...
├── apps/                  # 移动端/桌面应用
├── docs/                  # 文档
└── dist/                  # 构建输出
```

### 1.2 技术栈

- **运行时**: Node.js 22+, TypeScript (ESM)
- **构建**: tsdown, tsc
- **测试**: Vitest
- **HTTP/WebSocket**: Hono, ws
- **AI 集成**: @anthropic-ai/vertex-sdk, @modelcontextprotocol/sdk
- **终端 UI**: @clack/prompts, osc-progress

---

## 2. 核心架构原理

### 2.1 入口与启动流程

入口文件 `openclaw.mjs` 的启动流程：

```
openclaw.mjs
  → 检查 Node.js 版本 (需要 22.12+)
  → 加载 dist/entry.js
  → src/index.ts:runLegacyCliEntry()
  → src/cli/run-main.ts:runCli()
  → 构建命令行程序
```

**关键设计：**
- **双模式入口**：作为 CLI 运行时执行命令；作为库导入时导出 API
- **延迟加载**：命令按需注册，优化启动速度

### 2.2 Gateway 服务器

Gateway 是 OpenClaw 的核心运行时服务。

**主要职责：**
- WebSocket 服务器，处理客户端连接
- 管理消息通道 (Telegram, Discord, WhatsApp, Signal 等)
- 运行 AI Agent 会话
- 处理认证、配置热加载、健康监控

**关键组件：**

| 文件 | 职责 |
|-----|------|
| `src/gateway/server.impl.ts:201` | Gateway 主服务器 |
| `src/gateway/server-channels.ts` | 通道管理器 |
| `src/gateway/server-methods.ts` | RPC 方法处理 |
| `src/gateway/auth.ts` | 认证授权 |

### 2.3 插件系统

OpenClaw 采用**微内核 + 插件**架构。

**插件类型：**

1. **Channel Plugins** (`src/channels/plugins/`): 消息通道插件
   - 定义了 `ChannelPlugin` 接口
   - 支持消息收发、状态管理、配置向导

2. **Provider Plugins**: AI 模型提供商插件
   - 认证管理、模型选择、流式响应

3. **Extension Plugins** (`extensions/`): 独立扩展包
   - 例如：`msteams`, `matrix`, `zalo`, `voice-call`, `feishu`

**插件 SDK** (`src/plugin-sdk/`):
- 提供标准化的 API 接口
- 通过 `openclaw/plugin-sdk/*` 导出
- 支持 channel-runtime、gateway-runtime、agent-runtime 等

**关键文件：**
- `src/plugin-sdk/channel-contract.ts`: Channel 插件契约
- `src/plugin-sdk/channel-runtime.ts`: 运行时支持
- `src/plugins/runtime/index.ts`: 插件运行时

### 2.4 消息路由系统

路由系统决定消息如何分发到正确的 Agent。

**核心概念：**
- **Session Key**: 会话唯一标识，格式：`{agentId}:{channel}:{accountId}:{peerKind}:{peerId}`
- **Binding**: 将特定聊天绑定到指定 Agent
- **Account**: 通道账户隔离

**路由解析流程** (`src/routing/resolve-route.ts`):

```
1. 检查 peer 绑定 → binding.peer
2. 检查 guild/角色绑定 → binding.guild+roles
3. 检查 team 绑定 → binding.team
4. 检查 account 绑定 → binding.account
5. 使用默认 Agent → default
```

### 2.5 Agent 运行时

Agent 是与 AI 模型交互的核心。

**关键组件：**
- `src/agents/agent-command.ts`: Agent 命令处理
- `src/agents/pi-embedded.ts`: 嵌入式 Pi Agent 运行时
- `src/agents/model-selection.ts`: 模型选择逻辑
- `src/agents/auth-profiles.ts`: 认证配置管理

**Agent 执行流程：**

```
用户消息 → Gateway 接收
  → 路由解析
  → 会话加载
  → 模型选择
  → 调用 AI Provider
  → 流式响应处理
  → 消息发送
```

### 2.6 ACP (Agent Control Protocol)

`src/acp/` 目录实现了 Agent 控制协议：

- 会话管理、控制平面
- 子 Agent 派发
- 持久化绑定

---

## 3. 消息处理流程

### 3.1 整体流程图

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Telegram   │────▶│   Channel    │────▶│   Routing   │
│  Discord    │     │   Plugin     │     │   Engine    │
│  WhatsApp   │     └──────────────┘     └──────┬──────┘
│  Signal     │                                  │
│  iMessage   │                                  ▼
│  Web        │     ┌──────────────────────────────────┐
│  Matrix     │     │         Gateway Server           │
│  Teams      │     │  ┌─────────────┐                 │
└─────────────┘     │  │   Session   │    ┌─────────┐  │
                    │  │   Manager   │───▶│  Agent  │  │
                    │  └─────────────┘    │ Runtime │  │
                    │                     └────┬────┘  │
                    │                          │       │
                    │                     ┌────▼────┐  │
                    │                     │  AI     │  │
                    │                     │ Provider│  │
                    │                     │(Claude/ │  │
                    │                     │ OpenAI) │  │
                    │                     └─────────┘  │
                    └──────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Reply Pipeline │
                    │  (Markdown →    │
                    │   Channel Format)│
                    └────────┬────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Send to User   │
                    └─────────────────┘
```

### 3.2 Agent 入口调用链

```
用户消息
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Channel Plugin (Telegram/Discord/WhatsApp...)              │
│     接收消息，转换为 MsgContext                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. dispatchInboundMessage()                                    │
│     src/auto-reply/dispatch.ts:35                               │
│     - 创建 ReplyDispatcher                                      │
│     - 调用 dispatchReplyFromConfig()                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. dispatchReplyFromConfig()                                   │
│     src/auto-reply/reply/dispatch-from-config.ts:124            │
│     - 检查消息去重                                               │
│     - 解析会话存储                                               │
│     - 触发 hooks (message_received, inbound_claim)              │
│     - 调用 getReplyFromConfig()                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. getReplyFromConfig()                                        │
│     src/auto-reply/reply/get-reply.ts                           │
│     - 解析命令/消息内容                                           │
│     - 构建回复调度器                                              │
│     - 调用 runReplyAgent()                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. runReplyAgent()                                             │
│     src/auto-reply/reply/agent-runner.ts                        │
│     - 准备 agent 运行参数                                        │
│     - 调用 runAgentTurnWithFallback()                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. runAgentTurnWithFallback()  ⭐ 关键入口                      │
│     src/auto-reply/reply/agent-runner-execution.ts:77           │
│                                                                 │
│     根据配置选择执行路径:                                         │
│     ├─ runCliAgent()        → 外部 CLI (如 claude CLI)          │
│     │   src/agents/cli-runner.ts:53                             │
│     │                                                           │
│     └─ runEmbeddedPiAgent() → 内嵌 Pi Agent (默认)              │
│         src/agents/pi-embedded-runner/run.ts:267                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. runEmbeddedPiAgent()  ⭐ 最终 Agent 运行入口                  │
│     src/agents/pi-embedded-runner/run.ts:267                    │
│                                                                 │
│     核心流程:                                                    │
│     1. 解析会话车道 (SessionLane)                        │
│     2. 加载运行时插件                                             │
│     3. 运行 hooks (before_model_resolve, before_agent_start)    │
│     4. 解析模型 resolveModelAsync()                              │
│     5. 准备 SessionManager                                       │
│     6. 构建系统提示词                                             │
│     7. 调用 AI Provider API                                      │
│     8. 流式处理响应                                               │
│     9. 发送回复到通道                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 关键入口文件

| 文件路径 | 行号 | 函数名 | 职责 |
|---------|------|--------|------|
| `src/auto-reply/dispatch.ts` | 35 | `dispatchInboundMessage()` | 入站消息分发入口 |
| `src/auto-reply/reply/dispatch-from-config.ts` | 124 | `dispatchReplyFromConfig()` | 配置驱动的回复调度 |
| `src/auto-reply/reply/agent-runner.ts` | - | `runReplyAgent()` | Agent 运行准备 |
| `src/auto-reply/reply/agent-runner-execution.ts` | 77 | `runAgentTurnWithFallback()` | **核心：选择执行引擎** |
| `src/agents/pi-embedded-runner/run.ts` | 267 | `runEmbeddedPiAgent()` | **最终：内嵌 Agent 运行** |
| `src/agents/cli-runner.ts` | 53 | `runCliAgent()` | 外部 CLI Agent 运行 |

---

## 4. 飞书 Channel 实现

### 4.1 消息接收流程

```
飞书用户发送消息
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│  飞书服务器                                                    │
│  (推送消息事件)                                                │
└───────────────────────────┬───────────────────────────────────┘
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
        ▼                                       ▼
┌───────────────────────┐           ┌───────────────────────┐
│  Webhook 方式         │           │  WebSocket 方式       │
│  (飞书主动推送)        │           │  (长轮询订阅)         │
│                       │           │                       │
│  POST /feishu/events  │           │  Lark WS Client       │
│  → HTTP Server        │           │  → wsClient.start()   │
└───────────┬───────────┘           └───────────┬───────────┘
            │                                   │
            └───────────────┬───────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  monitor.transport.ts                                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  monitorWebhook() 或 monitorWebSocket()                  │  │
│  │                                                          │  │
│  │  1. 接收 HTTP/WebSocket 请求                             │  │
│  │  2. 验证签名 (x-lark-signature)                          │  │
│  │  3. 处理 challenge 响应 (首次配置)                        │  │
│  │  4. 调用 eventDispatcher.invoke(payload)                 │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  monitor.account.ts: registerEventHandlers()                  │
│                                                               │
│  注册事件处理器:                                               │
│  - im.message.receive_v1  → 消息接收                          │
│  - reaction.created       → 表情回应                          │
│  - card.action.triggered  → 卡片交互                          │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  bot.ts: handleFeishuMessage()  ⭐ 核心消息处理入口            │
│  extensions/feishu/src/bot.ts:221                             │
│                                                               │
│  处理流程:                                                     │
│  1. 消息去重检查 (finalizeFeishuMessageProcessing)             │
│  2. 解析消息内容 parseFeishuMessageEvent()                     │
│  3. 解析发送者名称 resolveFeishuSenderName()                   │
│  4. 群组权限检查 isFeishuGroupAllowed()                        │
│  5. DM 配对验证 (dmPolicy: pairing/allowlist/open)             │
│  6. 解析路由 resolveAgentRoute()                               │
│  7. 解析媒体内容 resolveFeishuMediaList()                      │
│  8. 构建 MsgContext buildCtxPayloadForAgent()                  │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  reply-dispatcher.ts: createFeishuReplyDispatcher()           │
│  extensions/feishu/src/reply-dispatcher.ts:95                 │
│                                                               │
│  创建回复调度器:                                               │
│  - 设置打字指示器 (typingIndicator)                            │
│  - 配置消息发送 (sendMessageFeishu)                            │
│  - 处理流式回复 (streaming-card)                               │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  core.channel.reply.dispatchReplyFromConfig()                 │
│  bot.ts:1122                                                  │
│                                                               │
│  进入核心消息分发流程 → Agent 运行                             │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 关键入口文件

| 文件路径 | 行号 | 函数名 | 职责 |
|---------|------|--------|------|
| `extensions/feishu/src/monitor.ts` | 31 | `monitorFeishuProvider()` | 启动飞书监听 |
| `extensions/feishu/src/monitor.transport.ts` | 84, 129 | `monitorWebSocket()`, `monitorWebhook()` | 接收飞书推送 |
| `extensions/feishu/src/monitor.account.ts` | 254 | `registerEventHandlers()` | 注册事件处理 |
| `extensions/feishu/src/bot.ts` | 221 | `handleFeishuMessage()` | **核心消息处理** |
| `extensions/feishu/src/reply-dispatcher.ts` | 95 | `createFeishuReplyDispatcher()` | 创建回复调度器 |

### 4.3 传输方式对比

| 特性 | Webhook | WebSocket |
|-----|---------|-----------|
| 启动函数 | `monitorWebhook()` | `monitorWebSocket()` |
| 需要公网 IP | ✅ 是 | ❌ 否 |
| 连接方式 | 飞书主动推送 | 客户端长连接 |
| 适用场景 | 生产环境 | 开发/内网环境 |
| 签名验证 | HTTP headers | 内置 |

### 4.4 配置示例

```yaml
channels:
  feishu:
    appId: "cli_xxx"
    appSecret: "xxx"
    encryptKey: "xxx"          # 用于签名验证
    webhookPort: 3000          # Webhook 端口
    webhookPath: "/feishu/events"
    dmPolicy: "pairing"        # DM 策略: pairing/allowlist/open
    groupPolicy: "allowlist"   # 群组策略
    allowFrom: ["ou_xxx"]      # 允许的用户/群组
```

---

## 5. Agents 人设存储机制

### 5.1 配置存储位置

```
~/.openclaw/                          # 状态目录
├── openclaw.json                     # 主配置文件 (包含 agents 定义)
├── agents/                           # 多 agent 数据目录
│   ├── default/                      # 默认 agent
│   │   └── agent/                    # agentDir
│   │       ├── AGENTS.md            # Agent 行为指南
│   │       ├── SOUL.md              # 核心人设/性格
│   │       ├── IDENTITY.md          # 身份信息
│   │       ├── USER.md              # 用户信息
│   │       ├── TOOLS.md             # 工具使用指南
│   │       ├── BOOTSTRAP.md         # 引导文件
│   │       ├── HEARTBEAT.md         # 心跳提示
│   │       ├── MEMORY.md            # 记忆存储
│   │       └── .openclaw/           # agent 状态目录
│   │           ├── sessions/        # 会话数据
│   │           └── auth-profiles.json # 认证配置
│   └── assistant/                    # 其他 agent (如 assistant)
│       └── agent/
│           └── ...
└── workspace/                        # 默认工作目录
```

### 5.2 Agent 配置结构

**`src/config/types.agents.ts:61-95`** 定义：

```typescript
type AgentConfig = {
  id: string;                    // Agent ID
  default?: boolean;             // 是否为默认 agent
  name?: string;                 // 显示名称
  workspace?: string;            // 工作目录
  agentDir?: string;             // Agent 数据目录

  // 人设配置
  identity?: IdentityConfig;     // 身份配置

  // 其他配置
  model?: AgentModelConfig;      // 模型配置
  skills?: string[];             // 技能过滤
  humanDelay?: HumanDelayConfig; // 人性化延迟
  sandbox?: AgentSandboxConfig;  // 沙箱配置
  tools?: AgentToolsConfig;      // 工具配置
};

type IdentityConfig = {
  name?: string;    // Agent 名称
  theme?: string;   // 卡片主题色
  emoji?: string;   // 头像 emoji
  avatar?: string;  // 头像图片路径/URL
};
```

### 5.3 Workspace Bootstrap 文件

**`src/agents/workspace.ts:25-32`** 定义了人设相关的文件：

| 文件名 | 用途 | 说明 |
|-------|------|------|
| `AGENTS.md` | Agent 行为指南 | 多 agent 协作说明 |
| `SOUL.md` | **核心人设** | Agent 的性格、风格、价值观 |
| `IDENTITY.md` | 身份信息 | Agent 的身份、背景 |
| `USER.md` | 用户信息 | 用户的偏好、背景 |
| `TOOLS.md` | 工具指南 | 工具使用说明 |
| `BOOTSTRAP.md` | 引导文件 | 初始化引导提示 |
| `HEARTBEAT.md` | 心跳提示 | 心跳检查时的响应 |
| `MEMORY.md` | 记忆存储 | 长期记忆 |

### 5.4 Bootstrap 文件加载流程

```
Agent 运行时
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  resolveBootstrapContextForRun()                        │
│  src/agents/bootstrap-files.ts:98                       │
│                                                         │
│  1. 解析 workspaceDir                                   │
│  2. 加载 bootstrap 文件                                 │
│  3. 应用 hook 覆盖                                      │
│  4. 构建上下文文件                                      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  loadWorkspaceBootstrapFiles()                          │
│  src/agents/workspace.ts:487                            │
│                                                         │
│  按顺序加载:                                            │
│  1. AGENTS.md                                          │
│  2. SOUL.md       ← 核心人设                            │
│  3. TOOLS.md                                           │
│  4. IDENTITY.md   ← 身份信息                            │
│  5. USER.md                                            │
│  6. HEARTBEAT.md                                       │
│  7. BOOTSTRAP.md                                       │
│  8. MEMORY.md                                          │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  buildSystemPrompt()                                    │
│  src/agents/system-prompt.ts                            │
│                                                         │
│  将 bootstrap 文件内容注入到系统提示词中                │
└─────────────────────────────────────────────────────────┘
```

### 5.5 配置示例

**`~/.openclaw/openclaw.json`:**

```json5
{
  "agents": {
    "defaults": {
      "model": "claude-sonnet-4-6",
      "provider": "anthropic"
    },
    "list": [
      {
        "id": "default",
        "default": true,
        "name": "OpenClaw",
        "identity": {
          "name": "OpenClaw",
          "emoji": "🤖",
          "theme": "blue",
          "avatar": "https://example.com/avatar.png"
        },
        "model": "claude-sonnet-4-6",
        "workspace": "~/projects/workspace"
      },
      {
        "id": "assistant",
        "name": "研究助手",
        "identity": {
          "name": "Research Bot",
          "emoji": "🔬",
          "theme": "green"
        },
        "agentDir": "~/.openclaw/agents/assistant/agent",
        "model": "gpt-4o"
      }
    ]
  }
}
```

**`~/.openclaw/agents/default/agent/SOUL.md` (核心人设):**

```markdown
# Soul

You are OpenClaw, a helpful AI assistant.

## Personality
- Friendly and professional
- Concise but thorough
- Proactive in suggesting solutions

## Communication Style
- Use clear, simple language
- Break down complex topics
- Provide examples when helpful
```

### 5.6 人设生效优先级

```
┌─────────────────────────────────────────────────────────┐
│  1. IdentityConfig (openclaw.json)                      │
│     - name, emoji, theme, avatar                        │
│     → 用于 UI 显示和回复格式                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. SOUL.md / IDENTITY.md (workspace bootstrap)         │
│     - 核心人设和身份定义                                 │
│     → 注入到系统提示词中                                 │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. Hooks 覆盖                                          │
│     - before_agent_start hook 可动态修改                │
│     → 运行时动态调整                                     │
└─────────────────────────────────────────────────────────┘
```

### 5.7 多 Agent 隔离

每个 Agent 通过独立的目录实现隔离：

- **`agentDir`**: 每个 agent 有独立的数据目录
- **Session 隔离**: 会话数据按 `agent:<id>:...` 格式的 sessionKey 分离
- **Auth 隔离**: 每个 agent 可配置独立的认证配置 (`auth-profiles.json`)
- **Memory 隔离**: 每个 agent 有独立的 `MEMORY.md`

### 5.8 关键代码路径

| 功能 | 文件 | 函数 |
|-----|------|------|
| Agent 配置解析 | `src/agents/agent-scope.ts:118` | `resolveAgentConfig()` |
| Identity 解析 | `src/agents/identity.ts:6` | `resolveAgentIdentity()` |
| Workspace 初始化 | `src/agents/workspace.ts:311` | `ensureAgentWorkspace()` |
| Bootstrap 加载 | `src/agents/workspace.ts:487` | `loadWorkspaceBootstrapFiles()` |
| 系统提示词构建 | `src/agents/system-prompt.ts` | `buildSystemPrompt()` |
| 配置文件路径 | `src/config/paths.ts:106` | `resolveCanonicalConfigPath()` |

---

## 6. 关键设计模式

### 6.1 依赖注入

通过 `createDefaultDeps()` 创建可测试的依赖：

```typescript
// 允许在测试中注入 mock 依赖
const deps = createDefaultDeps({
  fs: mockFs,
  config: mockConfig,
});
```

### 6.2 插件架构

微内核 + 可插拔扩展：

```typescript
// Channel 插件接口
interface ChannelPlugin {
  id: string;
  start(): Promise<void>;
  stop(): Promise<void>;
  handleInbound?(ctx: MsgContext): Promise<void>;
}
```

### 6.3 事件驱动

使用事件系统解耦模块：

```typescript
// Agent 事件
emitAgentEvent('session_start', { sessionKey, agentId });
emitAgentEvent('message_sent', { messageId, channel });

// Session 生命周期事件
emitSessionLifecycleEvent('session_created', session);
```

### 6.4 配置层叠

多层配置覆盖：

```
环境变量 → 运行时覆盖 → 配置文件 → 默认值
```

### 6.5 会话持久化

基于 JSON 文件的会话存储：

```typescript
// 会话文件路径
const sessionPath = resolveSessionFilePath(storePath, sessionKey);

// JSONL 格式存储
// 每行一个消息记录
```

---

## 附录：配置路径

| 路径 | 说明 |
|-----|------|
| `~/.openclaw/openclaw.json` | 主配置文件 |
| `~/.openclaw/agents/<id>/agent/` | Agent 数据目录 |
| `~/.openclaw/workspace/` | 默认工作目录 |
| `~/.openclaw/credentials/` | OAuth 凭证 |
| `~/.openclaw/sessions/` | 会话数据 |
| `OPENCLAW_CONFIG_PATH` | 配置文件路径环境变量 |
| `OPENCLAW_STATE_DIR` | 状态目录环境变量 |

---

*文档生成时间: 2026-03-21*