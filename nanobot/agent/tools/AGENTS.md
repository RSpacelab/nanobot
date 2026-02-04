# nanobot Agent 工具系统

## 概述
动态工具注册和执行系统，支持 Agent 在运行时调用各种工具。所有工具继承自 `Tool` 基类，通过 `ToolRegistry` 管理。

## 结构
```
nanobot/agent/tools/
├── base.py           # Tool 抽象基类
├── registry.py       # 动态工具注册表
├── filesystem.py     # 文件操作（读、写、编辑、列表）
├── shell.py          # Shell 命令执行
├── web.py            # Web 搜索和页面抓取
├── message.py        # 向聊天通道发送消息
└── spawn.py         # 子代理生成工具
```

## WHERE TO LOOK

| 任务 | 位置 | 说明 |
|------|--------|------|
| 工具注册表 | `registry.py` | `ToolRegistry` 类，管理工具的注册和执行 |
| 工具基类 | `base.py` | 所有工具继承的 `Tool` 抽象类 |
| 文件操作 | `filesystem.py` | ReadFileTool, WriteFileTool, EditFileTool, ListDirTool |
| Shell 执行 | `shell.py` | ExecTool（超时 60 秒，破坏性操作警告） |
| Web 工具 | `web.py` | WebSearchTool（Brave API）, WebFetchTool |
| 消息发送 | `message.py` | MessageTool（通过 bus 发送） |
| 子代理 | `spawn.py` | SpawnTool（通过 SubagentManager） |

## 约定

### 工具注册
- 所有工具必须继承自 `Tool` 基类
- 必须实现 `name` 和 `description` 属性
- 必须实现 `execute()` async 方法
- 通过 `ToolRegistry.register()` 动态注册

### 工具执行
- 工具执行结果作为字符串返回
- 通过 `ToolRegistry.execute()` 执行工具（自动调用 `tool.execute()`）
- 执行错误会被捕获并返回错误消息

### 工具定义格式
- `to_schema()` 方法将工具转换为 OpenAI 格式
- 包含 name, description, parameters（JSON Schema）

## 唯一样式

### 双异步队列模式
- Agent 不直接访问工具，而是通过 `ToolRegistry` 中介
- 工具执行结果通过队列返回给 Agent 循环

### 上下文注入
- `MessageTool` 和 `SpawnTool` 需要运行时上下文（channel, chat_id）
- 通过 `set_context()` 方法设置消息目标

## 注意事项
- 工具执行是异步的，必须在 async 函数中使用
- Shell 工具有破坏性，使用时需谨慎
- 工具结果会被添加到 Agent 消息历史中
