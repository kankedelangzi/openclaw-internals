# 第一章：架构总览

## 1.1 设计哲学

OpenClaw 的架构设计围绕一个核心问题：**如何让 AI 助手真正成为一个可扩展的运行时平台？**

市面上的 AI 应用大多是这样的结构：

```
应用代码 + LLM API + 固定功能
```

每次加新功能都要改核心代码，每次换平台都要重写适配层。OpenClaw 试图解决的是：

```
Gateway（核心不变）+ Skill（功能扩展）+ Channel（渠道适配）+ Node（算力扩展）
```

这四个维度都可以独立扩展而不影响其他部分。这就是 OpenClaw 的设计哲学：**最小核心 + 标准化扩展点**。

## 1.2 OpenClaw全景架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Gateway 进程                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     核心管理层                             │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐   │   │
│  │  │   Router   │  │  SessionMgr │  │   PluginMgr    │   │   │
│  │  │  消息路由   │  │   会话管理   │  │    插件管理     │   │   │
│  │  └────────────┘  └────────────┘  └────────────────┘   │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐   │   │
│  │  │ ChannelMgr │  │  SkillMgr   │  │    NodeMgr     │   │   │
│  │  │   渠道管理   │  │   技能管理   │  │    节点管理     │   │   │
│  │  └────────────┘  └────────────┘  └────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     扩展层                                │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│  │  │ Telegram │ │  Feishu  │ │  Discord │ │ 飞书Wiki  │  │   │
│  │  │  Plugin  │ │  Plugin  │ │  Plugin  │ │  Plugin  │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   WebSocket API (18789)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                                    │
         │ WebSocket                          │ SSH/Tailscale
         ▼                                    ▼
┌─────────────────┐                ┌─────────────────┐
│  本地客户端       │                │   远程客户端      │
│  CLI / macOS App │                │   Web UI / CLI   │
└─────────────────┘                └─────────────────┘
```

### 核心模块职责

| 模块 | 职责 | 关键类/文件 |
|---|---|---|
| **Router** | 消息路由、Handler分发、规则匹配 | `Router.ts` |
| **SessionMgr** | 会话生命周期、上下文管理、长期记忆 | `SessionManager.ts` |
| **PluginMgr** | 插件生命周期管理（加载/启用/禁用） | `PluginManager.ts` |
| **ChannelMgr** | 统一接入各消息渠道 | `ChannelManager.ts` |
| **SkillMgr** | 技能注册、发现、执行 | `SkillManager.ts` |
| **NodeMgr** | 远程节点注册、任务调度、状态监控 | `NodeManager.ts` |

## 1.3 消息流全链路

消息在 OpenClaw 内部的完整流转路径：

```
[外部消息]
    │
    ▼
┌─────────────────┐
│   Channel       │  ← Telegram/飞书/Discord/WhatsApp...
│   Plugin        │  ← 协议转换：外部格式 → 内部标准化消息
└────────┬────────┘
         │ inbound (标准化消息)
         ▼
┌─────────────────┐
│   Router        │  ← 规则匹配 → 确定Handler
│   (路由层)       │
└────────┬────────┘
         │ route
         ▼
┌─────────────────┐
│   Hook:Before   │  ← 认证检查、消息过滤、速率限制
│   (前置钩子)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   SkillMgr      │  ← 查找匹配的Skill并执行
│   (技能层)       │
└────────┬────────┘
         │ execute()
         ▼
┌─────────────────┐
│   SessionMgr    │  ← 更新上下文、管理记忆
│   (会话层)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Hook:After    │  ← 响应处理、日志记录
│   (后置钩子)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Channel       │
│   Plugin        │  ← 内部响应 → 外部格式转换
└────────┬────────┘
         │
         ▼
[外部响应]
```

### 时序图

```
用户  →  Telegram  →  Gateway  →  Router  →  SkillMgr
  ↑                                            │
  │                                            │
  │               SessionMgr ← Hooks          │
  │                                            │
  └──────────── 响应消息  ←────────────────────┘
```

## 1.4 WebSocket Gateway 协议

OpenClaw 的控制平面使用 WebSocket 通信，默认端口 **18789**。

### 连接握手

客户端连接后发送的第一个帧必须是 `connect`：

```json
// 客户端 → Gateway
{
  "type": "req",
  "id": "req-001",
  "method": "connect",
  "params": {
    "clientId": "cli-xxx",
    "platform": "cli",
    "token": "gateway_token_if_set"
  }
}

// Gateway → 客户端
{
  "type": "res",
  "id": "req-001",
  "ok": true,
  "payload": {
    "hello": "ok",
    "snapshot": {
      "presence": [...],
      "health": {...}
    }
  }
}
```

### 请求/响应模型

```json
// 请求
{
  "type": "req",
  "id": "req-002",
  "method": "agent",
  "params": {
    "message": "帮我查一下天气",
    "session": "user:100008220896"
  }
}

// 响应
{
  "type": "res",
  "id": "req-002",
  "ok": true,
  "payload": {
    "runId": "run-abc123",
    "status": "accepted"
  }
}

// 服务端推送（流式事件）
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run-abc123",
    "delta": "好的，",
    "done": false
  }
}

// 最终结果
{
  "type": "res",
  "id": "req-002",
  "ok": true,
  "payload": {
    "runId": "run-abc123",
    "status": "done",
    "summary": "北京今天天气晴朗，温度15-25度..."
  }
}
```

### 事件类型

| 事件 | 说明 |
|---|---|
| `agent` | Agent执行中的流式输出 |
| `chat` | 新消息事件 |
| `presence` | 用户在线状态变化 |
| `health` | Gateway健康状态 |
| `heartbeat` | 心跳（30秒一次） |
| `cron` | 定时任务触发 |
| `shutdown` | Gateway关闭通知 |

## 1.5 扩展点设计

OpenClaw 提供了四层扩展点，每层都可以独立开发而不影响其他层：

### Channel Plugin（渠道扩展）

接入新的消息平台，只需要实现 Channel Plugin：

```typescript
// Channel Plugin 接口
interface ChannelPlugin {
  name: string;
  version: string;
  
  // 启动渠道连接
  start(): Promise<void>;
  
  // 停止渠道连接
  stop(): Promise<void>;
  
  // 发送消息到渠道
  send(message: OutboundMessage): Promise<void>;
  
  // 接收来自渠道的消息
  onMessage(handler: (msg: InboundMessage) => void): void;
}
```

已有 Channel Plugin：Telegram、飞书、Discord、Slack、WhatsApp、WebChat、Signal、iMessage 等。

### Skill（功能扩展）

在 `skills/` 目录添加一个 `SKILL.md` 即可：

```
skills/
├── hello/
│   └── SKILL.md       # 你好技能
├── weather/
│   └── SKILL.md       # 天气查询
└── _installed/        # 从ClawHub安装的Skill
```

SKILL.md 结构：

```markdown
---
name: weather
description: "查询天气"
---

# Weather Skill

## 功能
查任意城市的天气

## 使用
\`@助手 天气 北京\`
```

### Plugin（系统扩展）

更底层的扩展，需要实现 Plugin SDK：

```typescript
// Plugin 接口
interface Plugin {
  name: string;
  version: string;
  
  // 插件加载时调用
  onLoad(): void | Promise<void>;
  
  // 插件启用时调用
  onEnable(): void | Promise<void>;
  
  // 插件禁用时调用
  onDisable(): void | Promise<void>;
  
  // 插件卸载时调用
  onUnload(): void | Promise<void>;
}
```

### Node（算力扩展）

远程机器作为计算节点接入：

```yaml
nodes:
  enabled: true
  list:
    - name: gpu-server
      address: "192.168.1.100"
      port: 18789
      token: "node_token_here"
```

## 1.6 线程/并发模型

OpenClaw 基于 Node.js 构建，天然是单线程事件循环模型。

### 为什么选择 Node.js

- **异步 I/O**：OpenClaw 大部分是 I/O 密集型（网络请求、文件读写），Node.js 的异步 I/O 非常适合
- **生态丰富**：npm 上有大量可直接使用的包
- **跨平台**：一次开发，到处运行
- **轻量**：比 JVM 系语言占用资源少

### 并发策略

OpenClaw 的并发不靠多线程，而靠：

1. **Node.js 事件循环**：所有 I/O 操作都是异步非阻塞的
2. **async/await**：用同步写法写异步逻辑，代码清晰
3. **子进程**：对于重计算任务，调度到 Node 子进程或远程 Node 执行
4. **定时任务队列**：cron 任务独立调度，不阻塞主流程

```typescript
// 一个Skill执行示例
async function executeSkill(skill: Skill, ctx: ExecutionContext) {
  // 异步执行，不阻塞其他Skill
  const result = await skill.execute(ctx);
  
  // 更新Session（异步写入）
  await ctx.session.updateHistory(skill.name, result);
  
  return result;
}
```

### 高并发场景

单个 Gateway 实例可以处理上千个并发连接。但如果有更高要求：

- **水平扩展**：多个 Gateway 实例，通过负载均衡分发
- **Node 调度**：把重任务调度到远程节点执行
- **消息队列**：高并发场景下，用外部消息队列缓冲

## 1.7 源码导读路径

如果你想深入 OpenClaw 源码，推荐阅读顺序：

```
1. 入口：packages/openclaw/src/index.ts
       → 理解整个项目的入口和模块组织

2. Gateway核心：packages/openclaw/src/gateway/
       → Gateway.ts（主进程）
       → Router.ts（消息路由）
       → SessionManager.ts（会话管理）

3. Channel层：packages/openclaw/src/channels/
       → 选择你熟悉的Channel（如telegram/）读源码

4. Skill系统：packages/openclaw/src/skills/
       → SkillManager.ts（技能管理）
       → SkillLoader.ts（Skill加载逻辑）

5. 协议层：packages/openclaw/src/protocol/
       → WebSocket消息编解码
       → JSON Schema定义
```

## 1.8 下一步

第一章到此结束。你现在应该理解：

- OpenClaw 的设计哲学（最小核心 + 标准化扩展）
- 全景架构图和核心模块职责
- 消息流的完整链路
- WebSocket Gateway 协议
- 四层扩展点设计
- Node.js 并发模型

下一章我们将深入 **消息流与路由**，看看 Router 是如何决定每条消息该交给谁处理的：

→ [第二章：消息流与路由](chapters/02-message-flow.md)
