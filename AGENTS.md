# nanobot 项目知识库

**生成时间：** 2026-02-04
**提交：** (未指定)
**分支：** main

## 概述
轻量级个人 AI 助手框架（Python 3.11+），使用 ~4600 行代码实现。采用混合语言架构（Python + TypeScript 用于 WhatsApp 桥接器），使用 LiteLLM 支持多 LLM 提供商。

## 结构
```
nanobot/
├── nanobot/          # 核心包
│   ├── agent/       # 核心 Agent 逻辑（循环、上下文、内存、技能、子代理）
│   ├── channels/    # 消息通道（Telegram、WhatsApp）
│   ├── providers/    # LLM 提供商（OpenRouter、Anthropic 等）
│   ├── bus/          # 消息路由（异步队列）
│   ├── cron/         # 定时任务
│   ├── heartbeat/     # 主动唤醒（每 30 分钟）
│   ├── session/       # 会话管理
│   ├── config/        # 配置（Pydantic 模式）
│   ├── cli/           # 命令行界面
│   ├── skills/        # 内置技能（github、weather、tmux 等）
│   └── utils/         # 工具函数
├── bridge/          # TypeScript WhatsApp 桥接器（Node.js + Baileys）
├── workspace/        # 运行时工作空间（用户配置、内存）
└── case/            # 演示资源（GIF）
```

## WHERE TO LOOK

| 任务 | 位置 | 说明 |
|------|--------|------|
| 核心 Agent 循环 | `nanobot/agent/loop.py` | LLM ↔ 工具执行循环 |
| 工具注册表 | `nanobot/agent/tools/registry.py` | 动态工具管理 |
| 子代理管理 | `nanobot/agent/subagent.py` | 后台任务执行 |
| 通道管理器 | `nanobot/channels/manager.py` | 消息路由协调 |
| 配置加载 | `nanobot/config/loader.py` | `~/.nanobot/config.json` |
| 会话管理 | `nanobot/session/manager.py` | 对话历史持久化 |
| 技能加载器 | `nanobot/agent/skills.py` | 进阶技能加载（workspace 优先） |
| 消息总线 | `nanobot/bus/queue.py` | 通道-Agent 解耦 |
| CLI 入口点 | `nanobot/cli/commands.py` | typer 应用程序 |
| LLM 提供商 | `nanobot/providers/litellm_provider.py` | 多提供商支持 |

## 约定

### 配置
- **构建系统**：hatchling（替代标准 setuptools）
- **Linting**：Ruff，line-length = 100（默认为 88）
- **类型检查**：TypeScript 严格模式（bridge/tsconfig.json）
- **测试**：pytest + asyncio_mode = "auto"
- **模块系统**：ES modules（bridge/package.json, tsconfig.json）

### 代码风格
- **Python**：100 字符行限制
- **导入排序**：使用 Ruff "I" 规则
- **异步模式**：所有 I/O 操作均使用 async/await

### 技能格式
- YAML frontmatter（name, description, metadata）
- 优先级：workspace 技能 > 内置技能
- 元数据：install 指令、requires 检查

## ANTI-PATTERNS（本项目）

### 禁止模式

**子代理约束**（`nanobot/agent/subagent.py`）：
- 禁止发起对话或承接侧边任务
- 禁止向用户直接发送消息（无 message 工具）
- 禁止生成其他子代理

**Agent 上下文规则**（`nanobot/agent/context.py`）：
- 正常对话时不要调用 message 工具
- 直接回复文本，不要嵌套工具调用

**Agent 指南**（`workspace/AGENTS.md`）：
- 不要只将提醒写入 MEMORY.md（无法触发通知）- 使用 cron 代替

### 安全警告

**Shell 工具**（`nanobot/agent/tools/shell.py`）：
- 命令超时 60 秒，输出会被截断
- 使用警告：具有破坏性操作

**工具文档**（`workspace/TOOLS.md`）：
- 具有破坏性操作时使用警告

### 技能创建约束

**技能格式**（`nanobot/skills/skill-creator/SKILL.md`）：
- 禁止创建多余文档或辅助文件（仅 SKILL.md）
- 避免重复：信息应位于 SKILL.md 或引用文件中
- 避免深层嵌套引用（从 SKILL.md 起最多一层）
- 避免问太多问题（从最重要的问题开始）

### Telegram 通道

（`nanobot/channels/telegram.py`）：
- 避免匹配内部单词，如 some_var_name（用于斜体文本解析）

### Tmux 技能

（`nanobot/skills/tmux/SKILL.md`）：
- 避免空格（tmux 名称应简短无空格）

## 唯一样式

### 消息路由
- 使用双异步队列（inbound/outbound）进行通道-Agent 解耦
- 消息使用 dataclass：`InboundMessage`, `OutboundMessage`
- 会话键格式：`"{channel}:{chat_id}"`

### 工具系统
- 所有工具继承自 `Tool` 基类（`nanobot/agent/tools/base.py`）
- 通过 `ToolRegistry` 动态注册
- 工具结果作为字符串返回

### 子代理
- 轻量级 Agent 实例，无消息工具，无 spawn 工具
- 通过消息总线宣布结果
- 系统消息注入：`channel="system"`, `chat_id="{origin}:{chat_id}"`

### 技能加载
- 始终加载技能（metadata.always=true）
- 可用技能仅显示摘要（代理使用 read_file 加载）
- 要求检查：shutil.which() 用于二进制文件

### 配置管理
- Pydantic BaseSettings 带环境前缀 `NANOBOT_`
- API 密钥优先级：OpenRouter > Anthropic > OpenAI > Gemini > Zhipu > vLLM
- 工作空间路径：`~/.nanobot/workspace`

## 命令

```bash
# 初始化配置和工作空间
nanobot onboard

# 直接聊天
nanobot agent -m "你好"

# 交互式聊天
nanobot agent

# 启动网关
nanobot gateway

# 查看状态
nanobot status

# 定时任务
nanobot cron add --name "daily" --message "早上好！" --cron "0 9 * * *"
nanobot cron list
nanobot cron remove <job_id>

# 通道管理
nanobot channels status
nanobot channels login  # WhatsApp QR 扫码
```

## 注意事项

### 架构注意事项
- **混合语言**：Python 核心包含 TypeScript 桥接器（bridge/）
- **无 CI/CD**：未发现 .github/workflows、Dockerfile 或 Makefile
- **无 tests/ 目录**：pyproject.toml 引用 `testpaths = ["tests"]` 但目录不存在
- **workspace/ 已提交**：运行时状态文件被提交到 git（应被忽略）

### 构建要求
- **桥接器**：在打包前必须编译（在 bridge/ 中运行 `npm run build`）
- **打包**：hatchling 强制将 bridge/ 包含到 wheel 中

### 配置文件
- 主配置：`~/.nanobot/config.json`
- 技能模板：`workspace/skills/{name}/SKILL.md`
- 引导文件：`workspace/AGENTS.md`, `workspace/USER.md`, `workspace/SOUL.md`
- 内存：`workspace/memory/MEMORY.md`, `workspace/memory/YYYY-MM-DD.md`

### 需要修复
1. **创建 tests/ 目录**或从 pyproject.toml 中删除 testpaths
2. **将 workspace/ 添加到 .gitignore**或移至 templates/
3. **添加 CI/CD**：GitHub Actions 用于 linting 和测试
4. **添加 Docker**：用于容器化部署
