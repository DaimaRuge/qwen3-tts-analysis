# OpenHarness v0.1.5 技术架构与源码研读报告

> 分析日期：2026-04-09
> 项目：HKUDS/OpenHarness
> GitHub：https://github.com/HKUDS/OpenHarness
> ⭐ Stars: 1,147 | Fork: 114 | 最新版本: v0.1.5
> 源码规模：197 个 Python 文件，38 个模块目录，1.8MB 源代码

---

## 一、项目概述

OpenHarness 是由香港大学数据科学实验室（HKUDS）开发的开源 Python 项目，旨在提供一个**开放的 AI Coding Agent Infrastructure（Agent Harness 框架）**。其核心定位是：Model provides intelligence, harness provides hands——语言模型提供智能，框架提供操作能力。

**一句话概括**：OpenHarness 是 Claude Code 的开源 Python 实现，支持多后端、多智能体、多平台。

### 1.1 核心特性速览

- **10 大子系统**：引擎、工具、技能、插件、权限、钩子、命令、MCP、内存、任务
- **43 种工具**：文件 I/O、Shell、搜索、Web、MCP 等
- **54 条命令**：REPL 交互式命令
- **多后端支持**：Claude、Kimi、GLM、OpenAI、OpenRouter、DashScope、DeepSeek、Ollama、GitHub Copilot 等
- **多智能体编排**：Swarm 子系统支持团队生命周期、子进程/进程内双后端
- **双 UI**：React TUI + CLI

### 1.2 版本历史（2026）

| 日期 | 版本 | 里程碑 |
|------|------|--------|
| 2026-04-01 | v0.1.0 | 首次开源发布，完成 Harness 架构 |
| 2026-04-06 | v0.1.2 | 统一安装流程，ohmo 个人 Agent App |

---

## 二、系统架构总览

### 2.1 模块目录结构

```
openharness/
├── api/              # LLM API 客户端（多后端统一抽象）
├── auth/             # 认证授权管理（多 provider）
├── bridge/           # 外部 CLI 桥接（Claude CLI / Codex CLI）
├── channels/         # 消息通道（飞书、Slack、Discord、Telegram 等）
├── cli.py            # Typer CLI 入口
├── commands/         # 54 条 REPL 命令
├── config/           # 配置管理（settings、paths、schema）
├── coordinator/      # 智能体协调（多智能体编排）
├── engine/           # Agent Loop 核心引擎
├── hooks/            # 生命周期钩子（PreToolUse/PostToolUse）
├── keybindings/      # 键盘快捷键解析
├── mcp/              # MCP (Model Context Protocol) 客户端/服务器
├── memory/           # 持久化记忆（MEMORY.md、session）
├── output_styles/    # 输出样式
├── permissions/      # 权限系统（多级模式）
├── plugins/          # 插件系统
├── prompts/          # 系统提示词管理
├── sandbox/          # 沙箱适配
├── services/         # 后台服务（cron scheduler、session storage）
├── skills/           # 技能加载（.md 动态加载）
├── state/            # 状态管理
├── swarm/            # 多智能体 Swarm 系统（团队协作）
├── tasks/            # 后台任务管理
├── themes/           # TUI 主题
├── tools/            # 43 种工具实现
├── types/            # Pydantic 类型定义
├── ui/               # React TUI
├── utils/            # 工具函数
└── voice/            # 语音模式
```

---

## 三、核心引擎（Engine）

### 3.1 Agent Loop 架构

Engine 是整个框架的心脏，包含 4 个核心文件：

```
engine/
├── __init__.py        # RuntimeBundle 组合体
├── query.py           # 核心查询循环（324 行）
├── query_engine.py    # 查询引擎编排
├── messages.py         # 消息类型（ConversationMessage、ToolResultBlock）
├── stream_events.py    # 流式事件定义
└── cost_tracker.py   # Token 消耗追踪
```

**核心循环伪代码**（来自 `query.py`）：

```python
while True:
    # 1. Auto-compaction — 上下文压缩（自动摘要老消息）
    messages = await auto_compact_if_needed(messages, ...)
    
    # 2. 流式 API 调用
    async for event in api_client.stream_message(request):
        if isinstance(event, ApiTextDeltaEvent):
            yield AssistantTextDelta  # 流式文本
        if isinstance(event, ApiMessageCompleteEvent):
            final_message = event.message
    
    # 3. 工具调用（单工具顺序，多工具并发）
    if len(tool_calls) == 1:
        result = await _execute_tool_call(context, tc.name, tc.id, tc.input)
    else:
        results = await asyncio.gather(*[_run(tc) for tc in tool_calls])
    
    # 4. 结果注入消息，继续循环
    messages.append(ToolResultBlock(...))
```

### 3.2 工具执行流程（`_execute_tool_call`）

这是整个框架最关键的一条执行路径，分 6 步：

```
1. PreToolUse Hook
   └─→ 可阻断工具执行，返回 reason

2. 输入验证（Pydantic model_validate）

3. 权限检查（PermissionChecker.evaluate）
   ├─→ 读操作检查 allowed_paths
   └─→ 写操作检查 denied_commands

4. 工具执行（tool.execute）
   └─→ 传入 ToolExecutionContext（含 cwd、metadata）

5. 日志记录（耗时、错误、输出长度）

6. PostToolUse Hook
```

### 3.3 Auto-Compact（上下文压缩）

框架内置智能上下文管理，每轮开始前检查 token 数量：

- **微压缩（Micro-compact）**：清除旧工具结果内容（轻量）
- **全压缩（Full compaction）**：LLM 摘要旧消息（精确但慢）

### 3.4 错误处理与重试

```python
# API 错误分类处理
if "connect" in error.lower() or "timeout" in error.lower():
    yield ErrorEvent("Network error...")  # 网络错误友好提示
else:
    yield ErrorEvent("API error...")

# 流式重试事件
if isinstance(event, ApiRetryEvent):
    yield StatusEvent(f"Retrying in {event.delay_seconds:.1f}s (attempt {event.attempt + 1})")
```

---

## 四、工具系统（Tools）

### 4.1 架构设计

```
BaseTool (ABC)
├── name: str           # 工具名
├── description: str    # 描述
├── input_model        # Pydantic 模型（自动生成 API schema）
├── execute()          # 抽象方法
└── is_read_only()     # 读/写标记（用于权限判断）

ToolRegistry
├── register(tool)     # 注册工具
├── get(name)          # 按名查找
├── list_tools()       # 列出全部
└── to_api_schema()   # 生成 API schema 列表
```

### 4.2 工具分类（43 种）

| 类别 | 工具列表 |
|------|----------|
| 文件 I/O | FileRead, FileWrite, FileEdit, Glob, Grep |
| Shell | Bash |
| LSP | LSPSymbol, LSPGoToDefinition |
| Web | WebSearch, WebFetch |
| 智能体 | AgentTool, EnterWorktree |
| 消息 | SendMessage, AskUserQuestion |
| 任务 | TaskCreate, TaskUpdate, CronList, CronCreate |
| MCP | MCPTool, MCPAuth, RemoteTrigger |
| 其他 | NotebookEdit, Bash, Glob, Grep, Bash |

**设计亮点**：
- 所有工具输入使用 Pydantic BaseModel，支持自动验证
- 权限系统通过 `is_read_only()` 判断是否需要写权限
- 工具执行上下文包含 `cwd`（工作目录），所有路径操作相对化

---

## 五、技能系统（Skills）

### 5.1 动态加载机制

Skills 目录下的 `loader.py`（134 行）实现按需加载：

```python
class SkillLoader:
    def load_skill(self, skill_path: Path) -> Skill
    def discover_skills(self, cwd: Path) -> list[Skill]
    def load_skill_md(self, name: str) -> str  # 读取 .md 文件
    
# 技能文件格式：SKILL.md
# 内容包含：description, instructions, examples
```

**技能分类**：
- **内置技能**：`exploring`, `debugging`, `impact_analysis`, `refactoring`
- **生成技能**：`gitnexus analyze --skills` 自动检测代码区域生成

### 5.2 与 Claude Code 的兼容

技能格式兼容 Anthropic 的 `skills` 规范，可直接使用社区现有的 `.md` 技能文件。

---

## 六、权限系统（Permissions）

### 6.1 多级权限模式

```python
PERMISSION_MODES = (
    "default",        # 交互确认
    "acceptEdits",    # 自动接受编辑
    "bypassPermissions", # 跳过所有检查（dangerous）
    "plan",           # 仅允许规划工具
    "dontAsk",        # 不询问
)
```

### 6.2 权限检查器

```python
PermissionChecker.evaluate(
    tool_name,
    is_read_only=bool,
    file_path=str,    # 绝对路径
    command=str,       # shell 命令
) → Decision(allowed, requires_confirmation, reason)
```

**检查规则**：
- 路径规范化：相对路径 → 绝对路径 → 匹配规则
- 路径支持通配符和目录前缀匹配
- Shell 命令与路径规则分离独立检查

---

## 七、钩子系统（Hooks）

### 7.1 事件类型

```python
HookEvent = Literal[
    "PreToolUse",    # 工具执行前
    "PostToolUse",   # 工具执行后
    "PreToolCall",   # 工具调用前（含并行分支）
    "PostToolCall",  # 工具调用后
    "PreProcess",    # 消息处理前
    "PostProcess",   # 消息处理后
]
```

### 7.2 四种钩子执行器

| 类型 | 执行方式 | 阻塞能力 |
|------|----------|----------|
| **CommandHook** | 子进程 shell 命令 | ✅ 可配置 |
| **HttpHook** | HTTP POST 请求 | ✅ 可配置 |
| **PromptHook** | LLM 判断是否通过 | ✅ |
| **AgentHook** | 全功能 Agent 判断 | ✅（更深度） |

**钩子参数注入**：`$ARGUMENTS` 模板变量自动替换为 JSON payload。

### 7.3 生命周期流程

```
PreToolUse Hook
    ↓ (不阻断)
输入验证
    ↓
权限检查
    ↓
工具执行 ─────────────────→ PostToolUse Hook
    ↓                              ↓
结果返回 ←──────────────── 结果注入消息列表
```

---

## 八、多智能体系统（Swarm + Coordinator）

### 8.1 双层架构

OpenHarness 的多智能体系统分为两层：

**Coordinator（协调层）**：
- `agent_definitions.py`（975 行）：智能体定义 YAML 解析
- `coordinator_mode.py`（519 行）：协调检测与编排逻辑

**Swarm（执行层）**：
- `team_lifecycle.py`（910 行）：持久化团队管理（JSON 文件存储）
- `permission_sync.py`（1168 行）：权限跨智能体同步
- `in_process.py`（693 行）：进程内智能体执行
- `subprocess_backend.py`（153 行）：子进程智能体执行
- `mailbox.py`（522 行）：智能体间消息传递
- `registry.py`（410 行）：智能体注册表
- `worktree.py`（315 行）：工作目录隔离

### 8.2 团队模型

```
~/.openharness/teams/<name>/
├── team.json          # 团队元数据
├── members.json      # 成员列表
└── mail/             # 消息队列（mailbox）
```

### 8.3 双后端执行

| 后端 | 适用场景 | 隔离性 | 通信方式 |
|------|----------|--------|----------|
| In-Process | 同进程内多 Agent | 低（共享内存） | 直接函数调用 |
| Subprocess | 需要强隔离 | 高（独立进程） | stdin/stdout JSON-RPC |

---

## 九、内存与持久化（Memory）

### 9.1 多层次记忆

```
MEMORY.md          # 用户长期记忆（跨会话）
.session/          # 会话快照（可恢复）
AGENTS.md          # 项目上下文（per-repo）
CLAUDE.md          # Claude Code 兼容上下文
```

### 9.2 Session 管理

```python
SessionSnapshot:
    - session_id: UUID
    - messages: list[ConversationMessage]  # 完整对话历史
    - system_prompt: str
    - model: str
    - summary: str  # 自动生成摘要
```

支持 `oh --continue` 恢复上一个会话，`oh --resume <id>` 恢复指定会话。

---

## 十、认证与多后端（Auth）

### 10.1 Provider 抽象

```
ProviderProfile:
    label: str          # 显示名
    provider: str        # 运行时 provider (anthropic/openai/...)
    api_format: str      # API 格式 (anthropic/openai/copilot)
    auth_source: str     # 认证方式
    base_url: str        # API 端点
    default_model: str
    credential_slot: str  # 凭证存储槽
```

### 10.2 支持的后端

**Anthropic 兼容系列**（API 格式 = anthropic）：
Claude 官方、Moonshot/Kimi、Zhipu/GLM、阿里云

**OpenAI 兼容系列**（API 格式 = openai）：
OpenAI 官方、OpenRouter、DeepSeek、SiliconFlow、Groq、Ollama（本地）

**特殊桥接**：
- Claude CLI subscription → 读取 `~/.claude/.credentials.json`
- Codex CLI subscription → 读取 `~/.codex/auth.json`
- GitHub Copilot → OAuth device flow

### 10.3 Profile 隔离

每个 provider 可配置独立凭证（`credential_slot`），避免多后端共享同一 API key。

---

## 十一、CLI 架构

### 11.1 Typer 命令树

```
openharness (oh)
├── oh setup              # 交互式配置向导
├── oh [prompt]           # REPL 交互模式
├── oh -p "prompt"        # 单次打印模式
├── oh mcp list/add/remove
├── oh plugin list/install/uninstall
├── oh auth login/logout/status/switch
├── oh provider list/use/add/edit/remove
└── oh cron start/stop/list/toggle/history/logs
```

### 11.2 交互式 Setup 向导

`oh setup` 实现了完整的交互式配置流：
1. 选择 Provider 类型（claude-api / openai-compatible）
2. 选择/认证具体后端
3. 选择模型
4. 保存激活 Profile

---

## 十二、React TUI

### 12.1 架构

```
ui/
├── app.py              # run_repl() / run_print_mode()
├── backend_protocol.py # TUI 后端通信协议
└── frontend/           # React Terminal（独立 npm 项目）
```

前端使用 **React + Textual**（Python TUI 框架），通过自定义协议与后端通信。

---

## 十三、设计哲学与工程亮点

### 13.1 优点

1. **清晰的分层架构**：Engine → Tools → Hooks → Permissions，每层职责明确
2. **Pydantic 优先**：所有类型定义使用 Pydantic，运行时验证 + 自动 schema 生成
3. **Async-First**：全面使用 `asyncio`，工具并行执行，I/O 高效
4. **插件化设计**：工具、技能、插件、钩子均可热插拔
5. **多智能体扩展**：Swarm 系统支持进程内和子进程两种模式
6. **向后兼容**：支持 Claude Code 的 AGENTS.md、CLAUDE.md、skills 格式
7. **完善的 CLI**：Typer + questionary 实现友好的交互式 TUI

### 13.2 值得关注的工程细节

**工具路径规范化**：
```python
# query.py 中统一处理 file_path vs path 字段差异
def _resolve_permission_file_path(cwd, raw_input, parsed_input):
    # 优先查 raw_input["file_path"] / raw_input["path"]
    # 再查 parsed_input.file_path / parsed_input.path
    # 相对路径自动拼接 cwd 并 resolve
```

**Permission Checker 双向验证**：
- 路径规则对读/写操作均可配置
- 命令白名单/黑名单分离

**Hook 执行器的超时控制**：
```python
stdout, stderr = await asyncio.wait_for(
    process.communicate(),
    timeout=hook.timeout_seconds,
)
# 超时杀死进程，防止挂起
```

**Team 持久化到磁盘**：
每个团队的元数据存储在 `~/.openharness/teams/<name>/team.json`，支持多智能体协作的持久化状态。

---

## 十四、关键源码文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `engine/query.py` | 324 | Agent Loop 核心循环 |
| `coordinator/agent_definitions.py` | 975 | 智能体定义与 YAML 解析 |
| `commands/registry.py` | 1587 | 54 条命令注册表 |
| `swarm/permission_sync.py` | 1168 | 多智能体权限同步 |
| `swarm/team_lifecycle.py` | 910 | 团队持久化管理 |
| `coordinator/coordinator_mode.py` | 519 | 协调模式编排 |
| `tools/base.py` | ~80 | BaseTool ABC + ToolRegistry |
| `hooks/executor.py` | 242 | 钩子执行引擎 |
| `cli.py` | ~900 | Typer CLI 入口 |
| `auth/manager.py` | ~300+ | 多后端认证管理 |
| `memory/manager.py` | ~150 | 记忆管理 |

---

## 十五、总结

OpenHarness 是一套**设计精良、生产级别**的 AI Coding Agent 基础设施。其核心优势在于：

1. **架构清晰**：20+ 子系统各司其职，通过接口而非继承实现解耦
2. **Provider 无关**：通过统一的 API 抽象层支持任意兼容后端
3. **安全优先**：多级权限 + Pre/Post Hook 双重安全网
4. **多智能体**：Swarm 系统提供了完整的团队协作能力
5. **开发者友好**：完善的 CLI、交互式配置、热加载钩子

对于想构建自己的 Coding Agent 或深度定制 Claude Code 类工具的开发者，OpenHarness 是目前最值得研究的开源参考实现。

---

*报告生成时间：2026-04-09 03:00 CST*
*分析工具：OpenClaw 🐾*
