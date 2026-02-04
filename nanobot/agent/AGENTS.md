# nanobot Agent 核心逻辑

## 概述
Agent 核心处理引擎，实现 LLM ↔ 工具执行循环。支持消息路由、会话管理、技能加载和子代理后台任务。

## 结构
```
nanobot/agent/
├── loop.py           # 主 Agent 循环（LLM ↔ 工具执行）
├── context.py        # 上下文构建器（system prompt + 历史 + 技能）
├── memory.py         # 持久化内存（每日笔记 + 长期内存）
├── skills.py         # 技能加载器（workspace 优先）
├── subagent.py       # 子代理管理器（后台任务）
└── tools/            # 内置工具（见 tools/AGENTS.md）
```

## WHERE TO LOOK

| 任务 | 位置 | 说明 |
|------|--------|------|
| Agent 循环 | `loop.py` | 核心处理引擎，接收消息 → 构建上下文 → LLM → 工具执行 |
| 上下文构建 | `context.py` | 组装 system prompt、引导文件、内存、技能、历史 |
| 内存管理 | `memory.py` | MemoryStore，支持每日笔记（YYYY-MM-DD.md）和长期内存（MEMORY.md） |
| 技能加载 | `skills.py` | SkillsLoader，workspace 技能 > 内置技能 |
| 子代理 | `subagent.py` | SubagentManager，后台任务执行（无 message 工具） |

## 约定

### Agent 循环
- 最大迭代次数：默认 20 次
- 处理两种消息类型：用户消息（channel=cli/telegram/whatsapp）和系统消息（channel=system，子代理结果）
- 系统消息的 chat_id 格式：`"{origin_channel}:{origin_chat_id}"`

### 上下文构建
- 引导文件（按顺序加载）：AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md
- 技能加载：两层机制
  - 始终加载（metadata.always=true）：完整内容
  - 可用技能：仅摘要，Agent 使用 read_file 按需加载
- 内存上下文：长期内存 + 今日笔记

### 子代理约束
- 禁止发起对话或承接侧边任务
- 禁止向用户直接发送消息（无 message 工具）
- 禁止生成其他子代理
- 最多 15 次迭代
- 结果通过消息总线以系统消息形式注入回主 Agent

### 技能系统
- YAML frontmatter 解析（name, description, metadata）
- 要求检查：shutil.which() 检查二进制文件，os.environ 检查环境变量
- 元数据字段：install（安装指令）、requires（bins, env）、always（始终加载）、available（可用性）

## 唯一样式

### 消息路由解耦
- 通道和 Agent 通过双异步队列（MessageBus）解耦
- `InboundMessage`：从通道到 Agent
- `OutboundMessage`：从 Agent 到通道
- 会话键格式：`"{channel}:{chat_id}"`

### 子代理结果路由
- 子代理完成任务后，通过消息总线发送系统消息
- 主 Agent 处理系统消息，将结果路由回原始用户
- 子代理提示包含："Summarize this naturally for user. Keep it brief."

### 工具上下文管理
- `MessageTool` 和 `SpawnTool` 需要运行时上下文（channel, chat_id）
- 在处理每条消息时，通过 `set_context()` 设置目标

## 注意事项
- Agent 循环在网关模式下持续运行（gateway 命令）
- 直接模式使用 `process_direct()` 方法（cli/agent 命令）
- 系统消息用于子代理结果分发，也用于心跳和定时任务
