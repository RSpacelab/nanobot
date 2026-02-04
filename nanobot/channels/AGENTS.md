# nanobot 通道管理

## 概述
聊天通道协调器，管理 Telegram 和 WhatsApp 等多个通道，处理消息路由和分发。

## 结构
```
nanobot/channels/
├── manager.py    # 通道管理器（协调所有通道）
├── base.py       # BaseChannel 抽象接口
├── telegram.py   # Telegram 通道实现
└── whatsapp.py   # WhatsApp 通道实现（WebSocket 桥接）
```

## WHERE TO LOOK

| 任务 | 位置 | 说明 |
|------|--------|------|
| 通道管理器 | `manager.py` | 初始化通道、启动/停止、路由出站消息 |
| 基类 | `base.py` | BaseChannel 抽象接口（start, stop, send, is_running） |
| Telegram | `telegram.py` | python-telegram-bot 集成，支持文本和媒体 |
| WhatsApp | `whatsapp.py` | WebSocket 连接到 TypeScript 桥接器（ws://localhost:3001） |

## 约定

### 通道管理器
- 动态初始化：根据配置启用的通道创建实例
- 并发启动：所有通道并发启动
- 出站分发：后台任务轮询 outbound 队列并分发给对应通道
- 优雅关闭：取消分发任务，停止所有通道

### Telegram 通道
- 基于 `python-telegram-bot` 库
- 支持用户白名单：`allow_from` 列表
- 消息格式：`InboundMessage` → Agent，`OutboundMessage` → 用户
- 媒体处理：支持图片和文件附件

### WhatsApp 通道
- WebSocket 客户端，连接到独立的 TypeScript 桥接器
- 桥接器 URL：`ws://localhost:3001`（可配置）
- 支持电话号码白名单：`allow_from` 列表
- 媒体处理：支持图片、视频、文档

## 唯一样式

### 双队列解耦
- 入站消息：通道 → `bus.inbound` → Agent
- 出站消息：Agent → `bus.outbound` → 通道分发器 → 具体通道
- 通道订阅模式：通过 `subscribe_outbound()` 订阅特定通道的出站消息

### 会话键生成
- `InboundMessage.session_key` 属性：`f"{self.channel}:{self.chat_id}"`
- 用于 SessionManager 识别对话会话

### Telegram 斜体文本解析
- 避免匹配内部单词（如 `some_var_name`）
- 使用更精确的正则表达式

## 注意事项
- WhatsApp 需要 Node.js >= 18 和独立的桥接器进程
- `nanobot channels login` 命令启动桥接器 QR 扫码
- 桥接器编译：运行 `npm run build` 生成 `dist/index.js`
