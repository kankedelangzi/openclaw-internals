# 第一章：架构总览

## 1.1 OpenClaw全景图

OpenClaw的核心组件：

```
┌─────────────────────────────────────────┐
│           Gateway Process               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │Router   │  │SessionMgr│  │PluginMgr│ │
│  └─────────┘  └─────────┘  └─────────┘ │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ChannelMgr│ │ SkillMgr │  │ NodeMgr │ │
│  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────┘
           │           │
     ┌─────┴─────┐ ┌──┴───┐
     │ Channel   │ │ Node  │
     │ Plugins   │ │ Agents│
     └───────────┘ └──────┘
```

## 1.2 核心模块职责

| 模块 | 职责 |
| --- | --- |
| Router | 消息路由、根据规则派发给不同Handler |
| SessionMgr | 会话管理、上下文维护、长期记忆 |
| PluginMgr | 插件生命周期管理 |
| ChannelMgr | 多渠道消息统一接入 |
| SkillMgr | 技能注册、发现、执行 |
| NodeMgr | 远程节点调度和管理 |

## 1.3 消息流

```
User → Channel(Telegram/飞书等) → Gateway → Router → SkillMgr → execute()
                                                        ↓
                                                  SessionMgr
                                                        ↓
                                                        ↓
                                                    Channel → User
```

## 1.4 下一步

- [第二章：消息流](02-message-flow.md) - 消息在内部是怎么流转的
