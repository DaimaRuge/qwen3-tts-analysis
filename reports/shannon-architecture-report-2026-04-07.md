# 技术架构与源码研读报告：Shannon — AI 自动化渗透测试框架

**项目名称：** Shannon (by KeygraphHQ)  
**仓库地址：** https://github.com/KeygraphHQ/shannon  
**Stars：** 36,347 | **Forks：** 3,807  
**技术标签：** AI Pentester · White-box Security Testing · Autonomous Agent  
**报告日期：** 2026-04-07  
**分析深度：** 架构设计 · 核心模块 · 关键设计决策 · 源码树结构

---

## 一、项目概述与核心定位

Shannon 是 KeygraphHQ 开发的一款**自主式白盒 AI 渗透测试引擎**，用于 Web 应用和 API 的自动化安全评估。与传统 SAST 工具不同，Shannon 的核心理念是**"只报告可实际利用的漏洞"**（Proof-of-Concept driven），而非产生大量误报的理论风险。

### 1.1 为什么需要 Shannon？

现代团队通过 Claude Code、Cursor 等 AI 编程工具实现持续交付，但渗透测试仍是一年一度的活动。这种 364 天的安全盲区是 Shannon 试图填补的空白——让每一次代码变更都能触发一次按需渗透测试。

### 1.2 产品版本

| 版本 | 许可证 | 定位 |
|------|--------|------|
| **Shannon Lite** | AGPL-3.0 | 本地化 CLI 工具，扫描自有源码库 |
| **Shannon Pro** | 商业版 | 一体化 AppSec 平台：SAST + SCA + 秘钥检测 + 业务逻辑安全测试 + 自主渗透测试 |

> **本报告聚焦 Shannon Lite（开源版）**，Pro 版技术细节仅在对比时引用。

---

## 二、整体架构

### 2.1 Monorepo 组织结构

Shannon 采用 **pnpm workspace + Turbo** 构建 Monorepo，目录结构如下：

```
shannon/
├── apps/
│   ├── cli/           # 命令行入口包（npx 交互层）
│   └── worker/        # 核心工作引擎（Temporal Worker + 渗透测试逻辑）
├── packages/          # （预留，当前未直接可见）
├── .claude/
│   └── commands/      # Claude Code Agent 专用指令（debug/pr/review）
├── infra/             # Docker Compose 配置（Postgres + Temporal）
├── sample-reports/    # OWASP Juice Shop 漏洞报告样例
└── scripts/           # 构建脚本
```

**核心技术栈：**
- **语言：** TypeScript（为主）+ Go（安全工具层）+ Python（Schemathesis）+ Ruby（WhatWeb）
- **任务编排：** Temporal（分布式工作流引擎）
- **容器化：** Docker + Docker Compose
- **Monorepo 管理：** Turbo + pnpm
- **代码检查：** Biome（替代 ESLint/Prettier）
- **Agent 核心：** Anthropic Claude Code SDK (`@anthropic-ai/claude-code`)
- **浏览器自动化：** Playwright

### 2.2 系统组件交互图

```
用户 CLI
  │
  │  npx @keygraph/shannon start -u https://target -r /repo
  ▼
┌─────────────────────────────────────────────────────────┐
│  Docker Compose 基础设施                                  │
│  ┌──────────┐  ┌─────────────┐  ┌───────────────────┐ │
│  │Temporal   │  │  Temporal   │  │   PostgreSQL       │ │
│  │Server     │◄─┤  Worker     │──┤   (Workflow State) │ │
│  │:7233      │  │  Container  │  │                    │ │
│  └──────────┘  └──────┬──────┘  └────────────────────┘ │
│                       │                                │
│              ┌────────▼────────┐                       │
│              │ Shannon Worker  │                       │
│              │ ─────────────── │                       │
│              │ 1. Reconnaissance │                     │
│              │ 2. Code Analysis  │                     │
│              │ 3. Attack Planning│                     │
│              │ 4. Exploitation  │                     │
│              │ 5. Reporting     │                     │
│              └─────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Docker 镜像架构（多阶段构建）

Dockerfile 采用 **Builder + Runtime** 双阶段构建：

**Builder Stage：**
- 基于 Chainguard Wolfi（最小攻击面 Linux 发行版）
- 安装构建工具链：Go、Node.js 22、Python3、Ruby
- 安装安全工具：Nmap、Subfinder（Go）、WhatWeb（Ruby）、Schemathesis（Python）
- 编译 Node.js 应用（pnpm build）
- 安装 `@anthropic-ai/claude-code` 和 `@playwright/cli`

**Runtime Stage：**
- 仅保留运行时依赖（剥离构建工具链）
- 包含 Chromium 浏览器（Playwright 所需）
- 以非 root 用户 `pentest` 运行
- 关键路径：`/app`（应用代码）、`/tmp`（会话数据）、`/tmp/.claude`（Claude Code 技能）

---

## 三、核心模块深度解析

### 3.1 CLI 模块（`apps/cli`）

CLI 是用户直接交互的入口，负责：
1. **参数解析**：解析 `-u`（目标 URL）、`-r`（仓库路径）、`-w`（workspace 名）、`-c`（YAML 配置）等
2. **基础设施管理**：调用 `docker compose` 启动/停止 Temporal + PostgreSQL
3. **Worker 容器生命周期**：拉取镜像、启动 ephemeral worker、挂载目标仓库
4. **日志聚合**：将容器日志实时流回终端

CLI 通过 `npx @keygraph/shannon` 分发，本质是调用预构建 Docker 镜像，用户无需克隆仓库。

### 3.2 Worker 模块（`apps/worker`）— 核心引擎

Worker 是整个系统的执行大脑。基于 **Temporal** 实现持久化工作流，确保渗透测试可以在任意时刻暂停/恢复。

#### 3.2.1 渗透测试工作流阶段

**阶段 1：侦察（Reconnaissance）**
- 调用 Nmap 探测网络拓扑
- 使用 Subfinder 进行子域名发现
- WhatWeb 指纹识别 Web 框架
- Schemathesis 对 API 进行基于属性的测试（Property-Based Testing）
- 扫描结果存入 Temporal Workflow State

**阶段 2：代码分析（Code Analysis）**
- 分析目标应用源码结构
- 识别入口点（API 路由、用户输入点）
- 绘制执行路径图
- **这是 Shannon Lite 与纯黑盒工具的本质区别**——白盒分析指导攻击策略

**阶段 3：攻击规划（Attack Planning）**
- 基于代码分析结果，LLM Agent 生成攻击向量候选列表
- 分类为 5 个攻击域：Injection、XSS、SSRF、Auth（认证）、Authz（授权）
- 优先级排序（可利用性 × 影响范围）

**阶段 4：并行利用（Parallel Exploitation）**
- **关键设计**：各攻击域并行执行，彼此独立
- 每个攻击域由独立的 Claude Code Agent 实例执行
- Agent 负责任务：
  - 登录/绕过认证（含 2FA/TOTP、SSO）
  - 浏览器自动化导航（Playwright）
  - 构造恶意请求（HTTP 参数、Header、Body）
  - 验证漏洞可利用性
- TOTP 生成器：`generate-totp.js`（支持时间型双因素令牌）
- 可交付物保存器：`save-deliverable.js`（存储 PoC 证据）

**阶段 5：报告生成（Reporting）**
- 仅包含**已证实可利用**的漏洞（不可利用的不报告，减少误报）
- 每个漏洞包含：CVSS 评分、完整 PoC（可 copy-paste 执行）、源码位置、修复建议
- 样例报告：OWASP Juice Shop 发现了 20+ 漏洞（含认证绕过和数据库泄露）

#### 3.2.2 Temporal 工作流设计

Temporal 的使用是架构亮点之一。相比直接用 setTimeout + 异步回调：
- **持久化状态**：工作流可在任意节点 crash-safe 重启
- **活动编排**：并行/串行任务依赖清晰声明
- **内置重试**：网络抖动、LLM API 限流自动重试
- **分形工作流**：每个攻击域可递归展开子攻击

### 3.3 Claude Code Agent 集成

Shannon 的渗透测试能力建立在 **Claude Code** 之上：
- 使用 `@anthropic-ai/claude-agent-sdk`（v0.2.38）
- Agent 运行在容器内的 `/tmp/.claude` 目录
- Playwright CLI 技能安装到 `/tmp/.claude/skills/playwright-cli`
- 支持 64K 输出 token（`CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000`）——处理大型漏洞报告

### 3.4 Shannon Pro 的核心技术——Code Property Graph（CPG）

Shannon Pro 的 SAST 引擎将代码转化为 **Code Property Graph**（代码属性图），融合：

| 图表层 | 内容 |
|--------|------|
| **AST** | 抽象语法树（代码结构） |
| **CFG** | 控制流图（程序执行路径） |
| **PDG** | 程序依赖图（数据依赖关系） |

**CPG 的价值：** 在数据流分析中，从 source（用户输入点）到 sink（危险函数调用）的路径可以被完整追踪。Shannon Pro 在每个路径节点上运行 LLM 来判断该点的 sanitization 是否充分——这是传统 SAST 工具无法实现的（传统工具依赖硬编码的安全函数白名单）。

---

## 四、关键设计决策分析

### 4.1 白盒 + 动态验证的结合

传统渗透测试工具（如 Burp Suite、OWASP ZAP）是纯黑盒的，依赖请求变异发现漏洞。Shannon 的白盒分析提供：
- **更精准的攻击面识别**：知道哪些参数对应源码中的哪些变量
- **业务逻辑漏洞发现**：黑盒工具无法理解的授权规则（如"跨租户访问"）
- **但最终仍需动态验证**：静态分析的理论漏洞必须被实际利用，才能进入报告

### 4.2 只报告"可利用"的漏洞

这是 Shannon 区别于传统 SAST 的最核心原则。传统工具会产生大量误报（理论风险但无法实际利用），导致开发者"告警疲劳"。Shannon 要求：
- 每个漏洞必须有完整 PoC
- 必须实际执行并成功
- 无法利用的不报告

### 4.3 多 Agent 并行架构

5 个攻击域（Injection、XSS、SSRF、Auth、Authz）完全并行执行，通过 Temporal 统一协调。这种架构：
- 天然横向扩展（可增加 Worker 实例）
- 隔离性好（一个域失败不影响其他域）
- 调度灵活（可按优先级分配 LLM 调用配额）

### 4.4 依赖 Chainguard Wolfi

选用 Chainguard Wolfi 而非 Alpine/Ubuntu，是**供应链安全**的考量：
- Wolfi 是为容器优化的最小化发行版
- 主动维护安全补丁
- Chainguard 提供 CVE 扫描和签名验证

### 4.5 自托管 Runner（Shannon Pro）

Shannon Pro 的企业部署模型（类 GitHub Actions self-hosted runner）将 LLM 调用和代码访问都放在客户网络内：
- 源码不离开客户基础设施
- 使用客户自己的 API Key
- Keygraph 控制面仅接收聚合后的漏洞报告

---

## 五、OWASP WSTG 覆盖率分析

根据 COVERAGE.md，Shannon 当前覆盖以下 WSTG 测试项（部分）：

| 测试类 | 覆盖情况 |
|--------|---------|
| Information Gathering | ✅ 5/9（WSTG-INFO-02, 06, 07, 08, 09, 10）|
| Configuration | ✅ 2/14（WSTG-CONF-01, 10）|
| Authentication | ✅ 10/11（WSTG-ATHN 全覆盖，仅 WSTG-ATHN-05, 06 未覆盖）|
| Authorization | ✅ 5/5（WSTG-ATHZ 全覆盖）|
| Session Management | ✅ 8/11（WSTG-SESS）|
| Input Validation | ✅ 10/20（Injection 类全覆盖）|
| Client-side | ✅ 7/14（WSTG-CLNT）|
| API Testing | ✅ 4/4（WSTG-APIT）|

**核心覆盖：** Injection、命令注入、XSS、SSRF、认证绕过、授权绕过

**未覆盖领域：** 错误处理测试、大部分加密测试（SCA/SAST 专用领域）

---

## 六、安全工具链集成

| 工具 | 用途 | 语言 |
|------|------|------|
| **Nmap** | 网络扫描、端口枚举 | C |
| **Subfinder** | 子域名发现 | Go |
| **WhatWeb** | Web 框架指纹识别 | Ruby |
| **Schemathesis** | API 属性测试 | Python |
| **Playwright** | 浏览器自动化 | TypeScript/Node.js |
| **Claude Code** | LLM Agent 核心 | TypeScript SDK |

---

## 七、工程化亮点

### 7.1 发布流水线

GitHub Actions 管理 beta/stable 双通道发布：
- `release-beta.yml`：发布 beta 版本到 npm `@shannon@beta`
- `release.yml`：发布正式版本
- `rollback-beta.yml` / `rollback.yml`：回滚机制

### 7.2 代码质量

- 使用 **Biome** 替代 ESLint + Prettier（统一工具，减少依赖）
- `pnpm --filter @shannon/worker run build` 实现精准包过滤构建
- Monorepo 的 onlyBuiltDependencies 优化 Docker 层缓存

### 7.3 环境配置

`.env.example` 支持：
- Anthropic API Key（推荐）
- Claude Code OAuth Token
- AWS Bedrock（ Anthropic 模型通过 AWS 路由）
- Google Vertex AI
- OpenRouter（实验性，支持 OpenAI/Gemini）

---

## 八、局限性与风险

1. **白盒限制**：必须能访问源码，纯黑盒场景不适用
2. **环境依赖**：需要 Docker 运行时，Windows 原生支持有限
3. **LLM 成本**：每次渗透测试涉及大量 LLM 调用，成本较高
4. **Agent 稳定性**：Claude Code Agent 在边界场景下的行为一致性仍有提升空间
5. **秘钥隔离**：Lite 版秘钥通过环境变量传递，有泄露风险

---

## 九、总结

Shannon 是一个**工程化程度极高**的 AI 安全工具，代表了 AI + Security 领域的重要趋势：

- **架构上**：Monorepo + Temporal 工作流 + Docker 容器化，现代化工程实践
- **理念上**："只报告可利用的漏洞"这一原则直接切中了传统安全工具的误报痛点
- **技术壁垒**：白盒代码分析 + LLM 推理 + 动态利用的组合，门槛较高
- **产品策略**：开源 Lite 版建立社区，Pro 版商业化，是经典的 OSS + SaaS 路径

Shannon 的核心创新在于**静态分析与动态验证的深度结合**——不满足于理论风险，而是用 LLM Agent 实际执行攻击来确认漏洞可利用性。这代表了安全测试工具从"规则匹配"到"智能体推理"的技术范式转变。

---

*报告生成工具：OpenClaw AI Assistant*  
*数据来源：GitHub Trending 2026-04-06 | KeygraphHQ/shannon 仓库*
