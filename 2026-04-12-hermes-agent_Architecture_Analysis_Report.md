# Hermes Agent 技术架构与源码研读报告

**项目**: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
** stars**: 65,000+ | **语言**: Python | **License**: MIT  
**分析日期**: 2026-04-12  
**分析人**: OpenClaw 自动架构分析系统

---

## 一、项目概述

Hermes Agent 是由 [Nous Research](https://nousresearch.com) 开发的自进化 AI Agent，主打"内置学习闭环"。与其他 AI Agent 不同，Hermes 的核心差异化在于：**它能从经验中创建技能、在使用中自我改进、主动持久化知识、跨会话检索历史对话，并持续构建对用户的深度认知模型**。

### 核心特性

| 特性 | 说明 |
|------|------|
| 多消息平台 | Telegram、Discord、Slack、WhatsApp、Signal、Email 等 15+ 平台 |
| 多终端后端 | local、Docker、SSH、Daytona、Singularity、Modal |
| 自学习闭环 | 技能自创建 + 使用中自改进 + 周期性记忆推送 |
| 多模型支持 | OpenRouter(200+模型)、Anthropic、MiniMax、OpenAI、Kimi、Nous Portal 等 |
| 记忆系统 | SQLite + FTS5 全文本搜索 + Honcho 用户建模 |
| MCP 集成 | 动态连接任何 MCP 服务器 |
| Cron 调度 | 自然语言任务调度，交付任意平台 |
| RL 研究支持 | Atropos RL 环境 + 轨迹压缩用于训练下一代工具调用模型 |
| OpenClaw 迁移 | 内置从 OpenClaw 导入 SOUL.md、MEMORY.md、Skills 等的迁移工具 |

---

## 二、系统架构总览

### 2.1 顶层入口与模块划分

```
┌──────────────────────────────────────────────────────────────┐
│                        Entry Points                          │
│  CLI (cli.py) | Gateway (gateway/run.py) | ACP | Batch | API │
└──────────────────┬───────────────────┬───────────────────────┘
                   │                   │
                   ▼                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    AIAgent (run_agent.py)                    │
│  ~9,200 行，核心对话循环                                       │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │   Prompt   │  │  Provider  │  │    Tool    │              │
│  │  Builder   │  │ Resolution │  │  Dispatch  │              │
│  └────────────┘  └────────────┘  └────────────┘              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │ Compression│  │  3 API     │  │   Tool     │              │
│  │ & Caching  │  │   Modes    │  │  Registry  │              │
│  │            │  │            │  │  (47工具)  │              │
│  └────────────┘  └────────────┘  └────────────┘              │
└──────────────────────────────────────────────────────────────┘
                   │                   │
                   ▼                   ▼
┌────────────────────────┐  ┌──────────────────────────┐
│     Session Storage     │  │      Tool Backends       │
│   (SQLite + FTS5)      │  │ Terminal(6) | Browser(5) │
│   hermes_state.py       │  │ Web(4) | MCP(dynamic)   │
└────────────────────────┘  └──────────────────────────┘
```

**三条核心数据流**:

1. **CLI 会话**: 用户输入 → HermesCLI.process_input() → AIAgent.run_conversation() → API调用 → 工具执行循环 → 持久化
2. **网关消息**: 平台事件 → Adapter.on_message() → 鉴权 → 会话解析 → AIAgent.run_conversation() → 响应
3. **Cron 任务**: 调度器 tick → 加载到期任务 → 创建无历史 AIAgent → 注入技能上下文 → 运行 → 交付

### 2.2 目录结构

```
hermes-agent/
├── run_agent.py          # AIAgent 核心对话循环 (~9,200 行)
├── cli.py                # HermesCLI 交互终端 UI (~8,500 行)
├── model_tools.py        # 工具发现、schema 收集、分发
├── toolsets.py           # 工具分组和平台预设
├── hermes_state.py       # SQLite 会话/状态数据库 + FTS5
├── hermes_constants.py   # HERMES_HOME, profile 路径解析
├── batch_runner.py       # 批量轨迹生成
│
├── agent/                # Agent 内部模块
│   ├── prompt_builder.py       # 系统提示词组装
│   ├── context_engine.py       # ContextEngine ABC (可插拔)
│   ├── context_compressor.py   # 默认压缩引擎 — 有损摘要
│   ├── prompt_caching.py       # Anthropic prompt caching
│   ├── auxiliary_client.py     # 辅助 LLM 客户端
│   ├── model_metadata.py       # 模型上下文长度 / token 估算
│   ├── anthropic_adapter.py    # Anthropic Messages API 格式转换
│   └── display.py             # KawaiiSpinner, 工具预览格式化
│
├── hermes_cli/           # CLI 子命令和设置
│   ├── main.py           # 入口点 (~5,500 行)
│   ├── config.py         # DEFAULT_CONFIG, 迁移逻辑
│   ├── commands.py        # COMMAND_REGISTRY 中心命令定义
│   ├── auth.py           # PROVIDER_REGISTRY, 凭证解析
│   ├── runtime_provider.py # Provider → api_mode + 凭证
│   ├── models.py         # 模型目录
│   ├── setup.py          # 交互式安装向导 (~3,100 行)
│   ├── skin_engine.py    # CLI 主题引擎
│   ├── plugins.py        # PluginManager 插件发现/加载
│   └── gateway.py        # hermes gateway 启停
│
├── tools/                # 工具实现
│   ├── registry.py      # 中心工具注册表
│   ├── approval.py       # 危险命令检测
│   ├── terminal_tool.py # 终端编排
│   ├── file_tools.py    # read_file, write_file, patch, search_files
│   ├── web_tools.py      # web_search, web_extract
│   ├── browser_tool.py  # 11 个浏览器自动化工具
│   ├── mcp_tool.py      # MCP 客户端 (~2,200 行)
│   └── environments/    # 终端后端 (local, docker, ssh, modal, daytona, singularity)
│
├── gateway/              # 消息平台网关
│   ├── run.py            # GatewayRunner 消息分发 (~7,500 行)
│   ├── session.py        # SessionStore 对话持久化
│   ├── delivery.py       # 出站消息投递
│   ├── pairing.py        # DM 配对授权
│   ├── hooks.py          # Hook 发现和生命周期事件
│   └── platforms/       # 15 个适配器: telegram, discord, slack, whatsapp,
│                         #   signal, matrix, email, sms, dingtalk, feishu,
│                         #   wecom, weixin, bluebubbles, homeassistant, webhook
│
├── acp_adapter/          # ACP 服务器 (VS Code / Zed / JetBrains)
├── cron/                 # 调度器 (jobs.py, scheduler.py)
├── plugins/memory/       # 记忆提供者插件
├── plugins/context_engine/ # 上下文引擎插件
├── skills/               # 捆绑技能 (始终可用)
├── optional-skills/      # 官方可选技能 (需显式安装)
└── tests/               # Pytest 测试套件 (~3,000+ 测试)
```

---

## 三、核心子系统深度分析

### 3.1 AIAgent 核心循环 (run_agent.py)

AIAgent 是整个系统的核心编排引擎，约 **9,200 行**，负责从提示词组装到工具分发再到 Provider 故障切换的全部逻辑。

#### 两种调用接口

```python
# 简单接口 — 返回最终响应字符串
response = agent.chat("Fix the bug in main.py")

# 完整接口 — 返回包含消息、元数据、用量统计的 dict
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,   # 自动构建
    conversation_history=None,  # 自动从 session 加载
    task_id="task_abc123"
)
```

#### 三种 API 执行模式

| API 模式 | 用途 | 客户端 |
|---------|------|--------|
| `chat_completions` | OpenAI 兼容端点 (OpenRouter、大多数 Provider) | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` (Responses 格式) |
| `anthropic_messages` | Native Anthropic Messages API | `anthropic.Anthropic` |

模式解析优先级: 显式 `api_mode` 参数 > Provider 检测 > Base URL 启发 > 默认 `chat_completions`

#### Turn 生命周期

```
run_conversation()
  1. 生成 task_id
  2. 将用户消息追加到对话历史
  3. 构建或复用缓存的系统提示词 (prompt_builder.py)
  4. 检查 Preflight 压缩 (>50% 上下文)
  5. 从对话历史构建 API 消息
  6. 注入临时提示词层 (预算警告、上下文压力)
  7. 应用 prompt caching 标记 (Anthropic)
  8.发起可中断的 API 调用
  9. 解析响应:
     - 若有 tool_calls → 执行 → 追加结果 → 回到步骤 5
     - 若为文本响应 → 持久化 session → 必要时 flush memory → 返回
```

#### 可中断 API 调用

```
主线程                           API 线程
┌──────────────┐  ┌──────────────┐
│ 等待:        │──▶│ HTTP POST   │
│ - response   │  │   to provider│
│ - interrupt  │  └──────────────┘
│ - timeout    │
└──────────────┘
```

用户发送新消息、`/stop` 或收到信号时：API 线程被放弃（响应丢弃），Agent 可干净地处理新输入或关闭，**对话历史中不会注入部分响应**。

#### 工具执行：顺序 vs 并发

- 单个工具调用 → 直接在主线程执行
- 多个工具调用 → 通过 `ThreadPoolExecutor` 并发执行
- 标记为 `interactive` 的工具（如 `clarify`）强制顺序执行
- 结果按原始工具调用顺序重新插入历史（不论完成顺序）

#### 迭代预算与降级

- 默认 **90 次迭代**（`max_turns` 可配置）
- 父子 Agent 共享预算
- 70% 使用率：追加预算警告
- 90% 使用率：追加紧急警告，要求立即给出最终回复
- 主模型失败时：按 `fallback_providers` 列表顺序尝试降级

---

### 3.2 提示词系统 (agent/prompt_builder.py)

#### 设计原则：缓存层 vs 临时层分离

这是 Hermes 最重要的设计决策之一：

**缓存的系统提示词层**（稳定，可被 Provider 缓存）:
1. Agent 身份 — `SOUL.md`（或 `DEFAULT_AGENT_IDENTITY`）
2. 工具感知行为指导
3. Honcho 静态块
4. 可选系统消息
5. **冻结的 MEMORY 快照**
6. **冻结的 USER profile 快照**
7. Skills 索引
8. Context 文件（AGENTS.md, .cursorrules 等）
9. 时间戳 / 会话 ID
10. 平台提示

**API 调用时临时层**（不持久化，不污染缓存）:
- `ephemeral_system_prompt`
- Prefill 消息
- 网关派生的会话上下文覆盖

#### Context 文件优先级（只加载第一个匹配）

| 优先级 | 文件 | 搜索范围 |
|--------|------|---------|
| 1 | `.hermes.md` / `HERMES.md` | CWD 向上到 git root |
| 2 | `AGENTS.md` | CWD |
| 3 | `CLAUDE.md` | CWD |
| 4 | `.cursorrules` / `.cursor/rules/*.mdc` | CWD |

所有 context 文件都经过**安全扫描**（检测提示注入模式）和**截断**（上限 20,000 字符）。

---

### 3.3 工具系统 (tools/registry.py + model_tools.py)

#### 自注册模型

每个工具模块在 import 时调用 `registry.register()`，向 `ToolRegistry._tools` 单例字典注册自己。`model_tools.py` 负责发现和 schema 收集。

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema={...},           # OpenAI function-calling schema
    handler=handle_terminal,
    check_fn=check_terminal, # 可选：可用性检查
    requires_env=["SOME_VAR"],
    is_async=False,
    description="Run commands",
    emoji="💻",
)
```

#### 工具发现流程

```
model_tools.py 被 import
  → _discover_tools() 按顺序 import 所有工具模块
  → 每个模块调用 registry.register()
  → MCP tools: tools.mcp_tool.discover_mcp_tools()
  → Plugin tools: hermes_cli.plugins.discover_plugins()
```

错误处理：可选工具加载失败（如缺少 fal_client）只记录日志，不阻止其他工具加载。

#### 工具集 (Toolset) 解析

工具集是命名工具包，通过以下方式解析：
- 显式启用/禁用列表
- 平台预设（hermes-cli、hermes-telegram 等）
- 动态 MCP 工具集
- 专用工具集如 hermes-acp

#### 分发流程

```
Model response (tool_call)
  → run_agent.py agent loop
  → model_tools.handle_function_call()
  → [Agent级工具?] → 直接处理 (todo, memory, session_search, delegate_task)
  → [Plugin pre-hook]
  → registry.dispatch(name, args)
  → 查找 ToolEntry
  → [Async?] → _run_async() 桥接
  → [Sync?] → 直接调用
  → 返回结果字符串（或 JSON 错误）
  → [Plugin post-hook]
```

两层错误包装：确保模型始终收到格式良好的 JSON，从不抛出未处理异常。

#### 危险命令审批流程 (DANGEROUS_PATTERNS)

```python
DANGEROUS_PATTERNS = [
    (r"rm -rf", "Recursive delete"),
    (r"mkfs|dd of=/dev/", "Filesystem formatting"),
    (r"DROP TABLE|DELETE FROM .* WHERE", "SQL destructive operations"),
    (r"> /etc/", "System config overwrite"),
    ...
]
```

- 检测到危险命令 → 交互式确认（CLI 模式）或异步回调（网关模式）
- 可选辅助 LLM 自动审批低风险命令
- 审批结果按会话跟踪，避免重复提示

---

### 3.4 会话持久化 (hermes_state.py)

#### 数据库架构

```
~/.hermes/state.db (SQLite, WAL 模式)
├── sessions       # 会话元数据、token 计数、计费
├── messages       # 完整消息历史
├── messages_fts   # FTS5 虚拟表（全文搜索）
└── schema_version # 迁移状态追踪
```

#### Sessions 表核心字段

- `id`, `source`, `user_id`, `model`, `model_config`
- `parent_session_id`（会话血脉追踪，压缩触发分裂时设置）
- `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`
- `estimated_cost_usd`, `actual_cost_usd`, `billing_provider`, `billing_mode`
- `message_count`, `tool_call_count`

#### FTS5 全文搜索

FTS5 虚拟表通过三个触发器与 messages 表同步（INSERT/UPDATE/DELETE）。支持丰富查询语法：

```sql
-- 基础搜索
results = db.search_messages("docker deployment")

-- FTS5 语法
"exact phrase"     -- 精确短语
docker OR kubernetes -- OR 查询
python NOT java    -- 排除
deploy*            -- 前缀匹配
```

#### 写入竞争处理

多个 Hermes 进程（网关 + CLI 会话 + worktree agent）共享一个 `state.db`：

- 短 SQLite 超时（1秒，而非默认 30s）
- 应用层重试 + 随机抖动（20-150ms，最多 15 次）
- `BEGIN IMMEDIATE` 事务，在事务开始时暴露锁竞争
- 每 50 次成功写入执行一次 WAL checkpoint（PASSIVE 模式）

```python
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020
_WRITE_RETRY_MAX_S = 0.150
_CHECKPOINT_EVERY_N_WRITES = 50
```

#### 会话血脉 (Session Lineage)

通过 `parent_session_id` 链形成会话链。当上下文压缩触发会话分裂时，新会话的 `parent_session_id` 指向原会话，允许追踪完整的会话进化树。

---

### 3.5 消息网关 (gateway/run.py)

约 **7,500 行**，支持 15 个平台适配器的长运行时进程。

#### 核心组件

| 组件 | 职责 |
|------|------|
| `GatewayRunner` | 消息调度中心 |
| `SessionStore` | 对话持久化 |
| `Delivery` | 出站消息投递 |
| `Pairing` | DM 配对授权 |
| `Hooks` | Hook 发现和生命周期事件 |
| `Status` | Token 锁，profile 作用域进程追踪 |

#### 平台适配器

支持平台: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Email, SMS, DingTalk, **飞书 (Feishu)**, WeCom, Weixin, BlueBubbles, Home Assistant, Webhook

#### Cron 子系统

任务存储在 JSON 中，支持多种调度格式，可附加技能和脚本，交付到任意平台。

```
Scheduler tick
  → 加载到期的 cron jobs
  → 创建无历史的 fresh AIAgent
  → 注入附加的技能作为上下文
  → 运行 job prompt
  → 交付响应到目标平台
  → 更新任务状态和 next_run
```

---

### 3.6 插件系统

三种发现源：
- `~/.hermes/plugins/`（用户插件）
- `.hermes/plugins/`（项目插件）
- pip entry points

两类专用插件：
- **Memory Provider**（`plugins/memory/`）：可插拔的记忆提供者，替换默认的 MEMORY.md/USER.md 方式
- **Context Engine**（`plugins/context_engine/`）：可插拔的上下文压缩引擎，替换默认的 `context_compressor.py`

两者均为单选配置，通过 `hermes plugins` 或 `config.yaml` 激活一个。

---

## 四、设计原则总结

| 原则 | 实践 |
|------|------|
| **提示词稳定性** | 系统提示词在对话中不变，不因缓存破坏性变更（除非显式 `/model` 等用户操作） |
| **可观测执行** | 每个工具调用通过 callbacks 对用户可见；CLI 显示 spinner，网关显示进度消息 |
| **可中断** | API 调用和工具执行可被用户输入或信号中途取消 |
| **平台无关核心** | 一个 `AIAgent` 类服务 CLI、Gateway、ACP、Batch、API Server；平台差异在入口点而非 Agent 本身 |
| **松耦合** | 可选子系统（MCP、plugins、memory providers、RL environments）使用注册表模式和 `check_fn` 门控，而非硬依赖 |
| **Profile 隔离** | 每个 profile（`hermes -p <name>`）有独立的 HERMES_HOME、config、memory、sessions、gateway PID |

---

## 五、与 OpenClaw 的对比

Hermes Agent 与 OpenClaw 有很多相似的设计哲学，这也解释了为什么 Hermes 内置了从 OpenClaw 迁移的工具：

| 维度 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| 核心架构 | AIAgent 类 + 工具注册表 | 几乎相同设计 |
| 记忆 | MEMORY.md / USER.md | 相同 + Honcho dialectic user modeling |
| Skills | SKILL.md 驱动 | Skills 系统 + agentskills.io 开放标准 |
| 平台支持 | 主要飞书/Telegram | 15+ 平台，更广泛 |
| 存储 | 文件系统 | SQLite + FTS5（更成熟） |
| 自进化 | 有限 | 内置学习闭环（自创建技能 + 使用中自改进） |
| RL 训练 | 无 | 内置 Atropos RL 环境 |
| 学习来源 | Nous Research（2025） | 独立开源（2026 爆火） |

**最大差异**：Hermes 的"内置学习闭环"是其他 Agent 框架所没有的——它不仅执行任务，还从任务中提取、固化和改进技能。

---

## 六、关键技术亮点

### 6.1 文件依赖链（工具注册时机）

```
tools/registry.py  (无依赖 — 所有工具文件 import 它)
  ↑
tools/*.py         (每个在 import 时调用 register())
  ↑
model_tools.py     (import registry + 触发工具发现)
  ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

这意味着**工具注册发生在任何 agent 实例创建之前**。添加新工具只需在 `model_tools.py` 的 `_discover_tools()` 列表中加一行 import。

### 6.2 Prompt Cache 友好架构

通过将稳定层（SOUL.md、skills、memory 快照）与临时层（ephemeral_system_prompt、gateway overlays）严格分离，Hermes 确保了系统提示词的稳定前缀，这使得 Anthropic 的 prompt caching 标记能发挥最大效果。

### 6.3 多进程写入竞争处理

通过短超时 + 随机抖动 + `BEGIN IMMEDIATE` + 定期 WAL checkpoint 的组合，巧妙规避了 SQLite 在多进程写入场景下的" convoy effect"问题。

### 6.4 向后兼容的工具集名称

旧工具集名称（带 `_tools` 后缀，如 `web_tools`）通过 `_LEGACY_TOOLSET_MAP` 映射到现代名称，确保历史配置不会因重命名而失效。

---

## 七、源码研读建议路径

```
1. 本文档 → 架构概览
2. run_agent.py → AIAgent 核心循环 (~9,200 行)
3. agent/prompt_builder.py → 提示词组装逻辑
4. hermes_state.py → SQLite 存储 + FTS5
5. tools/registry.py + model_tools.py → 工具注册与分发
6. gateway/run.py → 消息网关 (~7,500 行)
7. agent/context_compressor.py → 上下文压缩
8. cron/ → 调度器
```

---

## 八、总结

Hermes Agent 是一个**工程化程度极高**的 AI Agent 框架。它没有发明新概念，而是在现有优秀设计（OpenAI tool-calling、Anthropic prompt caching、LangChain式的工具注册表）基础上，做了大量**生产级打磨**：

- 65k stars 的爆发式增长反映的是真实需求
- 三种 API 模式支持任意 Provider，零锁定
- SQLite + FTS5 的会话存储比纯文件更可靠
- 多平台网关让 Agent"活在用户所在之处"
- 内置学习闭环让 Agent 真正超越"每次会话归零"的局限
- 与 OpenClaw 的迁移兼容性体现了务实的设计态度

对于想构建生产级 AI Agent 系统的开发者，Hermes 是一个值得深入研究的**工程标杆**。

---

*报告由 OpenClaw 每日代码架构分析系统自动生成*
*分析时间: 2026-04-12 19:00 UTC*
