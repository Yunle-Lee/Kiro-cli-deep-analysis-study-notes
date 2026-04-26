# 🔍 Kiro CLI — AI Agent 终端工具深度分析报告

> 分析日期：2026-04-26
> 分析对象：`kiro-cli`（Amazon Q Developer CLI 的衍生版本）

---

## 一、总览

**kiro-cli** 是一个用 **Rust** 编写的 AI Agent 命令行工具，是 **Amazon Q Developer CLI**（前身为 CodeWhisperer CLI/Fig）的分支/定制版本。它是一个运行在终端里的智能体（AI Agent），功能与我（当前这个 AI）非常相似：能读写文件、执行命令、对话聊天、编写和调试代码。

---

## 二、可执行文件结构

系统中有 **3 个二进制文件**，各自分工不同：

| 文件 | 类型 | 大小 | 职责 |
|------|------|------|------|
| `kiro-cli` | ELF 64-bit 可执行（含 debug_info） | 较大 | **主程序**：CLI 框架、认证、配置、自动补全、遥测、会话管理 |
| `kiro-cli-chat` | ELF 64-bit 可执行（含 debug_info） | 更大 | **聊天/Agent 引擎**：对话管理、工具调用、AI 模型交互、Agent 编排 |
| `kiro-cli-term` | ELF 64-bit 可执行（含 debug_info） | 中等 | **终端模拟器**：基于 alacritty_terminal，提供 TUI 模式的终端会话 |

> 三个都是 Rust 编译产物，未 strip 保留 debug_info，通过 `kiro.dev` 域名分发。

---

## 三、核心技术栈

| 类别 | 技术选型 |
|------|----------|
| **语言** | Rust（通过 Cargo 构建） |
| **异步运行时** | `tokio` |
| **CLI 解析** | `clap` v4 |
| **终端控制** | `crossterm` |
| **HTTP 客户端** | `hyper` + `reqwest` |
| **TLS** | `rustls`（纯 Rust 实现） |
| **HTTP/2** | `h2` |
| **WebSocket** | `tungstenite` |
| **数据库** | SQLite3（内嵌 libsqlite3） |
| **序列化** | `serde` / `serde_json` |
| **加密** | `ring` |
| **编码** | `encoding_rs` |
| **压缩** | `flate2` / `miniz_oxide` |
| **正则** | `regex` / `regex-automata` |
| **D-Bus** | `zbus`（Linux 桌面集成） |
| **终端模拟** | `alacritty_terminal`（kiro-cli-term 专用） |

---

## 四、系统架构

```
┌─────────────────────────────────────────────────┐
│                   kiro-cli                       │
│  ┌───────────┐ ┌──────────┐ ┌───────────────┐   │
│  │ CLI 入口   │ │ 认证系统  │ │ 自动补全引擎   │   │
│  │ (clap)     │ │ (OIDC)   │ │ (Shell 集成)  │   │
│  └───────────┘ └──────────┘ └───────────────┘   │
│  ┌───────────┐ ┌──────────┐ ┌───────────────┐   │
│  │ 配置管理   │ │ 遥测系统  │ │ 进程间通信     │   │
│  │ (SQLite)   │ │ (Cognito)│ │ (IPC/Remote)  │   │
│  └───────────┘ └──────────┘ └───────────────┘   │
└─────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│                 kiro-cli-chat                    │
│  ┌───────────┐ ┌────────────┐ ┌──────────────┐  │
│  │ Agent 编排 │ │ 工具系统    │ │ AI 模型交互   │  │
│  │ (ReAct)    │ │ (Tool Use) │ │ (LLM API)    │  │
│  └───────────┘ └────────────┘ └──────────────┘  │
│  ┌───────────┐ ┌────────────┐ ┌──────────────┐  │
│  │ 对话管理   │ │ 文件系统    │ │ MCP 协议      │  │
│  │ (SQLite)   │ │ 操作模块    │ │ 扩展          │  │
│  └───────────┘ └────────────┘ └──────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## 五、AI Agent 核心机制（ReAct 模式）

从二进制中提取的关键信息揭示了这套系统的工作方式：

### 5.1 Agent 循环

```
用户输入 → 思考(think) → 调用工具(tool_use) → 观察结果 → 再思考 → ...
```

关键字符串证据：
- `tool_use_id`、`tool_name`、`is_accepted`、`is_success`、`is_custom_tool`
- `output_token_size`、`custom_tool_call_latency`
- `qactuateagent_thought_chunk`、`user_message_chunk`

### 5.2 可用工具列表

从二进制字符串中能推断出以下工具：

| 工具 | 猜测功能 |
|------|----------|
| `ReadFile` / `FsRead` | 读取文件 |
| `WriteFile` / `FsWrite` | 写入文件 |
| `FileEdit` / `Edit` | 编辑文件（替换/插入） |
| `FileSearch` / `Search` / `Grep` | 搜索文件内容 |
| `BashCommand` / `Bash` | 执行 Shell 命令 |
| `Code` / `CodeTool` | 代码生成/分析（支持 Tree-sitter + LSP） |
| `WebSearch` / `Fetch` | 网页搜索和抓取 |
| `TodoList` / `Todo` | 任务列表管理 |
| `List` / `ListDir` | 列出目录 |
| `Think` / `Introspect` | 自我推理和元认知 |
| `Glossary` / `Notion` | 知识库/术语表 |

### 5.3 Agent 自定义系统

支持通过 **AGENTS.md** 配置文件（JSON Schema 校验）创建自定义 Agent：
- `/agent create` — AI 辅助或手动创建新 Agent
- `/agent edit` — 修改已有 Agent
- `/agent generate` — 自动生成 Agent 配置
- `/agent delete`、`/agent list`、`/agent set-default`
- `/agent swap` — 切换 Agent
- 支持 `preToolUse` 和 `postToolUse` 钩子函数

### 5.4 工具权限系统

- `/tools` 命令：精细化控制每个工具的权限
- `/tools trust-all` / `/tools untrust`
- 支持按 Agent 粒度控制工具启用/禁用

### 5.5 MCP（Model Context Protocol）集成

- 支持 MCP 服务器（`mcpServerInit`）
- 通过 MCP 协议扩展第三方工具

### 5.6 ACP（Agent Client Protocol）

从字符串 `acp`、`acp_agent`、`Agent Client Protocol (ACP)` 可知支持 Agent 间的客户端协议通信。

---

## 六、数据库设计（SQLite）

文件位置：`~/.local/share/kiro-cli/data.sqlite3`

### 6.1 表结构

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `migrations` | 数据库版本迁移记录 | version, migration_time |
| `history` | Shell 命令历史记录 | command, shell, cwd, exit_code, duration |
| `auth_kv` | OAuth 认证凭据存储 | key, value（OIDC token/device registration） |
| `state` | 应用状态持久化 | key, value（BLOB） |
| `conversations` | 对话存储（旧版 v1） | key, value |
| `conversations_v2` | 对话存储（新版 v2） | key, conversation_id, value, created_at, updated_at |

### 6.2 认证机制

- 采用 **OAuth 2.0 Device Authorization Grant**（设备码流程）
- 后端对接：`codewhisperer:completions`、`codewhisperer:analysis`、`codewhisperer:conversations` 三个作用域
- 支持多种身份源：
  - **Builder ID**（免费 AWS 身份）
  - **IAM Identity Center**（企业 SSO）
  - **External IdP**（第三方身份提供商）
  - **Social Login**（社交账号登录）
- 令牌刷新：自动检测过期并刷新 access_token
- 区域：`us-east-1`

### 6.3 遥测系统

- 通过 AWS Cognito 身份池认证
- 遥测客户端 ID：`telemetryClientId`
- 事件类型包括：`chatStart`、`chatEnd`、`tool_use`、`completionInserted`、`codeGeneration` 等
- 支持通过环境变量 `Q_DISABLE_TELEMETRY` 禁用

---

## 七、Shell 集成与自动补全

- 支持 **Bash**、**Zsh**、**Fish** 自动补全
- 支持 Fig 补全规范（`fig` 命令）
- Shell 钩子：`preexec`、`precmd` 函数拦截
- 内联 Shell 补全（inline completion）
- WSL 检测与提示

---

## 八、与我的（KiLee）对比

kiro-cli 和我（当前 AI Agent）在架构上有许多相似之处：

| 特性 | kiro-cli | 我（KiLee） |
|------|----------|-------------|
| **语言** | Rust | -（由宿主机系统提供） |
| **工具集** | ReadFile, WriteFile, Bash, Code, Search, Web, Todo | fs_read, fs_write, execute_bash, web_search, think |
| **循环模式** | ReAct（Reason-Act-Observe） | 同样使用 ReAct 循环 |
| **思维工具** | `Think` / `Introspect` | `think` |
| **文件操作** | FsRead + FsWrite + FileEdit | fs_read + fs_write |
| **命令执行** | Bash 命令 | execute_bash |
| **Web 搜索** | WebSearch / Fetch | web_search |
| **对话持久化** | SQLite 数据库 | -（由宿主管理） |
| **认证** | AWS OIDC OAuth2 | -（当前无认证） |
| **自定义 Agent** | AGENTS.md + schema | -（系统内置） |
| **MCP 扩展** | 支持 MCP 协议 | - |

---

## 九、关键设计亮点

1. **三进程架构**：将 CLI 框架、AI Agent 引擎、终端模拟器分离为三个独立二进制，职责清晰、可独立更新
2. **Rust 全栈**：从 HTTP 客户端到 TLS 到 SQLite，全部用 Rust 实现，无外部依赖
3. **Agent 自定义体系**：用户可以通过 AGENTS.md 配置文件创建自己的 Agent，支持 JSON Schema 校验
4. **权限精细控制**：工具级别的权限管理，安全可控
5. **多种认证方式**：支持个人身份、企业 SSO、第三方 IdP，灵活适配不同场景
6. **完整的遥测体系**：Product-led growth 思维，通过遥测数据优化产品
7. **文件操作带语义**：`fs_write` 工具支持 `summary` 参数，记录每次修改的目的

---

## 十、参考资料

- 官方仓库：`https://github.com/aws/amazon-q-developer-cli`
- 官方文档：`https://kiro.dev/docs/cli/`
- 原始项目 Forge：**Fig** → **Amazon Q Developer CLI** → **Kiro CLI**

---

## 十一、Agent 循环深度剖析（ReAct 引擎）

Agent 核心循环位于 `crates/agent/src/agent/agent_loop/mod.rs`，这是整个系统的"大脑"。

### 11.1 循环流程

```
User Input
    │
    ▼
┌─────────────────────────────────────────────┐
│  Agent Loop                                 │
│                                             │
│  ┌──────────┐    ┌──────────┐               │
│  │  思考     │───▶│  调用工具  │               │
│  │ (Think)   │    │ (Tool Use)│               │
│  └──────────┘    └────┬─────┘               │
│       ▲               │                     │
│       │               ▼                     │
│       │        ┌──────────┐                 │
│       │        │  观察结果  │                 │
│       │        │ (Observe) │                 │
│       │        └────┬─────┘                 │
│       │             │                       │
│       └─────────────┘                       │
│                                             │
│  流式响应处理：                               │
│  MessageStart → ContentBlockStart           │
│  → ContentBlockDelta → ContentBlockStop     │
│  → MessageStop / ToolUse                    │
└─────────────────────────────────────────────┘
```

### 11.2 流式响应协议

Agent 与 LLM 后端通信采用流式协议，事件类型包括：

| 事件 | 说明 |
|------|------|
| `MessageStartEvent` | 消息开始 |
| `ContentBlockStartEvent` | 内容块开始（包含 `tool_use_id`）|
| `ContentBlockDeltaEvent` | 内容块增量（流式文本块）|
| `ContentBlockStopEvent` | 内容块结束 |
| `MessageStopEvent` | 消息结束 |
| `MetadataEvent` | 元数据事件（用量统计）|
| `ContextUsageEvent` | 上下文用量（input_tokens / output_tokens / cache）|
| `ToolUseEvent` | 工具调用事件 |
| `MeteringEvent` | 计费事件 |

### 11.3 错误与重试

- **StreamErrorKind**：`StreamTimeout`, `Validation`, `Other`
- **服务端错误**：`ContextWindowOverflow`, `ServiceFailure`, `Throttling`, `Validation`
- 每次工具调用带唯一 ID：`tool_use_id`
- 支持 `max_tokens`、`stop_reason: EndTurn | MaxTokens | ToolUse | ContentFiltered`

---

## 十二、工具系统完整清单

从 `crates/agent/src/agent/tools/` 目录结构推断出完整工具列表：

### 12.1 内置工具（@builtin）

| 工具标识符 | Rust 模块 | 功能描述 | 参数/操作 |
|-----------|-----------|---------|----------|
| `fs_read` | `tools::fs_read` | 读取文件 | `limit`（行数）, `offset`（偏移） |
| `write` / `fs_write` | `tools::fs_write` | 写入文件 | `create`, `str_replace`, `insert`（三种模式） |
| `execute_bash` | `tools::execute_bash` | 执行 Bash 命令 | `command`, `timeout_ms` |
| `execute_cmd` | `tools::execute_cmd` | 执行命令（更通用） | 支持 `shell` 选项 |
| `shell` | — | Shell 命令别名 | 可配置不同的 shell |
| `ls` | `tools::ls` | 列出目录内容 | `depth`（递归深度）, 路径参数 |
| `grep` | `tools::grep` | 搜索文件内容 | `pattern`, `include`, `exclude`, `case_sensitive`, `output_mode`, `max_matches_per_file`, `max_files`, `max_total_lines` |
| `code` | — | 代码生成/分析 | 集成 Tree-sitter + LSP |
| `web_search` | — | 网页搜索 | 搜索查询, 返回标题/URL/摘要 |
| `web_fetch` | — | 抓取网页内容 | URL 参数 |
| `use_aws` / `aws` | — | AWS 服务操作 | 支持 S3 cp/mv/sync 等 |
| `image_read` | — | 读取图片 | 支持 GIF/PNG 格式 |
| `introspect` | — | 自我认知/元认知 | 回答关于自身的问题 |
| `knowledge` | — | 知识库操作 | `Add`, `Search`, `Remove`（BM25 检索） |
| `switch_to_execution` | — | 切换到执行模式 | 从对话模式切换到执行命令模式 |
| `use_subagent` | — | 调用子 Agent | `agentName`, `relevantContext` |
| `summary` | — | 文档总结 | 自动压缩对话历史 |
| `ls` | — | 目录列表 | 支持递归和文件信息显示 |

### 12.2 文件操作三种模式

`FsWrite` 文件写入支持三种操作模式：

```rust
enum FsWrite {
    FileCreate { content: String },          // 创建新文件
    StrReplace { old_str: String, new_str: String, replace_all: bool },  // 替换文本
    Insert { insert_line: u32 },             // 在指定行后插入
}
```

每次文件操作还附带 `__tool_use_purpose` 字段，记录修改目的。

### 12.3 工具调用格式

```json
{
  "tool_name": "fs_read",
  "input": {
    "path": "/path/to/file",
    "limit": 100,
    "offset": 0,
    "__tool_use_purpose": "Reading config file to understand project structure"
  }
}
```

---

## 十三、Agent 配置系统

### 13.1 Agent Schema

配置基于 JSON Schema：
- Schema 地址：`https://raw.githubusercontent.com/aws/amazon-q-developer-cli/refs/heads/main/schemas/agent-v1.json`
- Schema 版本：`AgentConfigV2025_08_22`

### 13.2 完整配置字段

```json
{
  "$schema": "https://raw.githubusercontent.com/aws/amazon-q-developer-cli/refs/heads/main/schemas/agent-v1.json",
  "prompt": "You are a custom agent...",
  "welcomeMessage": "Hello! I'm your custom agent.",
  "tools": ["fs_read", "fs_write", "execute_bash", "web_search"],
  "toolAliases": {},
  "toolsSettings": {
    "fsRead": { "allowedPaths": ["/src"], "deniedPaths": ["/src/secrets"] },
    "fsWrite": { "autoAllowReadonly": false },
    "executeCmd": {
      "allowedCommands": ["git", "npm", "cargo"],
      "deniedCommands": ["rm -rf"],
      "denyByDefault": true,
      "dangerous_options": ["--force"],
      "shells": ["/bin/bash"],
      "safe_commands": ["git status", "git diff"]
    },
    "useAws": {
      "allowedServices": ["s3", "ec2"],
      "deniedServices": ["iam"]
    }
  },
  "toolSchema": {},
  "hooks": {
    "preToolUse": { "command": "git stash", "timeout_ms": 5000 },
    "postToolUse": { "command": "git stash pop" }
  },
  "modelPreferences": {
    "costPriority": 0.3,
    "speedPriority": 0.3,
    "intelligencePriority": 0.4
  },
  "mcpServers": {
    "my-server": {
      "type": "local",
      "command": "node",
      "args": ["server.js"],
      "env": { "KEY": "value" }
    }
  },
  "allowedTools": ["fs_read", "grep"],
  "model": "claude-sonnet-4-20250514"
}
```

### 13.3 钩子系统（Hooks）

支持工具调用前后的自动化操作：
- `preToolUse` — 工具执行前运行钩子命令
- `postToolUse` — 工具执行后运行钩子命令
- 配置参数：`command`, `timeout_ms`, `max_output_size`, `cache_ttl_seconds`, `matcher`

### 13.4 技能系统（Skills）

Agent 可以加载来自不同来源的知识：
- `skill://.kiro/skills/*/SKILL.md` — 项目内技能
- `skill://~/.kiro/skills/*/SKILL.md` — 用户全局技能
- `.kiro` 文件 — 项目级配置
- `file://AmazonQ.md` — AmazonQ 规则
- `file://.amazonq/rules/**/*.md` — 项目目录规则

---

## 十四、MCP（Model Context Protocol）深度集成

### 14.1 MCP Server 配置

支持两种类型：

**本地 MCP Server**：
```json
{
  "command": "node",
  "args": ["server.js"],
  "env": { "KEY": "value" },
  "timeoutMs": 30000
}
```

**远程 MCP Server**：
```json
{
  "url": "https://mcp.example.com",
  "headers": { "Authorization": "Bearer xxx" },
  "oauthScopes": ["tools:read"],
  "timeoutMs": 30000
}
```

### 14.2 支持的 MCP 能力

| 能力 | 说明 |
|------|------|
| `tools` | 工具调用（`listChanged`, `cancel`） |
| `resources` | 资源访问（`subscribe`） |
| `prompts` | 提示模板 |
| `tasks` | 任务管理（`cancel`） |
| `sampling` | LLM 采样（`createMessage`） |
| `elicitation` | 诱导式交互（`UrlElicitation`, `FormElicitation`） |
| `roots` | 根目录管理 |
| `logging` | 日志记录 |

### 14.3 会话管理

- 使用 `Mcp-Session-Id` 头管理会话
- 支持 OAuth 认证流程
- 自动刷新令牌

---

## 十五、知识库系统

### 15.1 知识库操作

```rust
enum Knowledge {
    Add { content: String },                      // 添加知识
    Search {                                       // 搜索知识
        query: String,
        context_id: String,
        limit: u32,
        offset: u32,
        snippet_length: u32,
        sort_by: String,
        file_type: String,
    },
    Remove { context_id: String },                 // 移除知识
}
```

- 使用 BM25 检索算法（`BM25DataPoint`）
- 支持嵌入向量（`embedding_type`）
- 知识库配置：`source_path`, `item_count`, `auto_sync`

### 15.2 知识上下文

```rust
struct KnowledgeContext {
    source_path: String,
    item_count: u32,
    embedding_type: String,
    auto_sync: bool,
    created_at: u64,
    updated_at: u64,
    persistent: bool,
}
```

---

## 十六、权限与安全模型

### 16.1 文件系统权限

```rust
struct FileSystemPermissions {
    allowed_read_paths: Vec<String>,
    allowed_write_paths: Vec<String>,
    denied_read_paths: Vec<String>,
    denied_write_paths: Vec<String>,
}
```

### 16.2 运行时权限

- `RuntimePermissions` 控制工具调用是否可以继续
- 用户交互式授权："Tool was denied by user"
- 工具权限在 `agent::agent::permissions.rs` 中实现

### 16.3 命令执行安全

`ExecuteCmdSettings` 提供了精细的安全控制：
- `allowedCommands` — 允许的命令列表
- `deniedCommands` — 禁止的命令列表（优先级更高）
- `denyByDefault` — 默认拒绝所有命令（白名单模式）
- `dangerous_options` / `dangerous_env_vars` — 危险参数检测
- `safe_commands` / `safe_options` — 安全命令白名单
- `shells` — 允许的 Shell

---

## 十七、代码智能（Code Intelligence）

### 17.1 两种模式

1. **AST 模式**（Tree-sitter）：轻量级语法解析 + 模糊搜索，自动检测语言
2. **LSP 模式**（完整）：通过 Language Server Protocol 实现语义分析、导航、重构

### 17.2 LSP 支持的功能

- `GetCompletions` — 自动补全
- `GetDiagnostics` — 诊断信息
- `GotoDefinition` — 跳转到定义
- `FindReferences` — 查找引用
- `GetHover` — 悬停信息
- `GetDocumentSymbols` — 文档符号
- `RenameSymbol` — 符号重命名
- `SearchSymbols` — 符号搜索（支持正则）
- `PatternSearch` — 模式搜索
- `PatternRewrite` — 模式重写
- `SemanticTokens` — 语义高亮

### 17.3 代码库理解

- `condensed_repomap` — 压缩的仓库地图（类似 Aider 的 repo map）
- `files_processed`, `token_count`, `truncated`, `total_files`
- `prioritized_files`, `fci`, `mkloc` — 文件优先级排序
- `workspace_path`, `size_category` — 工作区分析

---

## 十八、对话管理

### 18.1 对话存储

使用 SQLite 的 `conversations_v2` 表：
- 主键：`(key, conversation_id)`
- 存储 JSON 格式的对话历史
- 包含 `created_at` 和 `updated_at` 时间戳

### 18.2 历史压缩

- `CompactStrategy` — 对话过长时的压缩策略
- 自动总结旧对话："Creating summary...", "truncating large messages"
- 压缩后迁移 `session_id`
- 最后 N 条消息保留原始格式

### 18.3 对话日志

```rust
enum LogEntryV1 {
    Prompt(String),           // 用户提示
    AssistantMessage(String), // AI 回复
    ToolResults(String),      // 工具调用结果
    Compaction(String),       // 压缩后的摘要
    ResetTo(String),          // 重置到某点
}
```

---

## 十九、子 Agent 系统

### 19.1 子 Agent 调用

```rust
struct SubagentInvocation {
    agent_name: String,
    relevant_context: String,
}
```

- `use_subagent` 工具允许主 Agent 调用子 Agent
- 子 Agent 共享上下文但独立执行
- 支持 `list_agents` 列出可用子 Agent
- 通过 `agentSpawn` 机制动态生成新 Agent

---

## 二十、Shell 集成详细机制

### 20.1 支持的 Shell

- Bash（补全 + 钩子）
- Zsh（补全 + `preexec`/`precmd` 钩子 + 自定义 `_complete` 函数）
- Fish（补全 + 语法高亮兼容）
- NuShell

### 20.2 Shell 钩子机制

```
用户输入命令 → preexec 钩子（记录命令开始）
         ↓
命令执行中...
         ↓
命令结束 → precmd 钩子（记录退出码、耗时）
         ↓
显示提示符 → prompt 钩子（显示 AI 建议）
```

### 20.3 补全触发

- 内联补全："inline.enabled"
- Fig 兼容模式
- 自动检测 WSL 环境
- 按键捕获（`crossterm`）

---

## 二十一、默认 Agent Prompt

从二进制中提取的默认系统提示：

```
"You are the default Kiro CLI agent, bringing the power of AI-assisted 
development directly to the user's terminal. You help with coding tasks, 
system operations, AWS management, and development workflows."
```

斜杠命令列表：`/help`, `/model`, `/agent`, `/clear`, `/quit`, `/usage`, `/paste`, `/tools`, `/plan`, `/feedback`, `/chat`, `/knowledge`, `/reply`

---

## 二十二、与我的完整对比（KiLee vs Kiro CLI）

| 维度 | KiLee（我） | Kiro CLI |
|------|------------|----------|
| **语言** | 由宿主系统驱动 | Rust 编译 |
| **宿主** | 终端内 AI Agent | ELF 64-bit 进程 |
| **工具数量** | ~5 个（think, fs_read, fs_write, execute_bash, web_search） | ~20+ 个工具 |
| **Agent 循环** | ReAct 循环 | ReAct 循环 + 流式流 |
| **文件系统操作** | fs_read / fs_write（create, str_replace, insert, append） | fs_read（支持 limit/offset）/ fs_write（create/str_replace/insert）+ ls + grep + mkdir |
| **搜索能力** | 基础版本（Search 模式） | 强大的 grep 工具 + 符号搜索 + 模式重写 |
| **代码理解** | 无 | Tree-sitter AST + LSP 语义分析 + repomap |
| **Web 搜索** | web_search | web_search + web_fetch |
| **知识库** | 无 | BM25 检索 + 嵌入向量 + 自动同步 |
| **子 Agent** | 无 | 支持子 Agent 调用和动态生成 |
| **MCP 协议** | 无 | 完整 MCP 支持（工具/资源/提示/任务/采样） |
| **钩子系统** | 无 | preToolUse / postToolUse 钩子 |
| **权限控制** | 无 | 细粒度文件系统 + 命令 + AWS 权限 |
| **认证** | 无 | AWS OIDC + Builder ID + IAM Identity Center + External IdP + Social |
| **持久化** | save_memory（简单） | SQLite（对话/配置/认证/遥测） |
| **自动补全** | 无 | Bash/Zsh/Fish 补全 + 内联补全 |
| **遥测** | 无 | AWS Cognito 事件追踪 |
| **自定义 Agent** | 无 | AGENTS.md + JSON Schema + AI-assisted 创建 |
| **对话历史** | 无 | SQLite 存储 + 自动压缩 |
| **协作模式** | 单一 Agent | 多 Agent + 子 Agent + MCP |
| **终端模拟** | 无 | 内置 alacritty 终端（kiro-cli-term） |

---

## 二十三、关键架构决策总结

1. **三进制分离**：CLI 入口、Agent 引擎、终端模拟器独立编译，降低耦合
2. **Rust 全栈无外部依赖**：从 HTTP/TLS 到 SQLite 全部 Rust 实现
3. **流式原生**：Agent 输出天然支持流式事件推送
4. **可扩展 Agent 体系**：JSON Schema 定义 + 技能系统 + MCP 协议
5. **安全优先**：命令白名单、路径权限、工具级别的允许/禁止
6. **AWS 生态深度绑定**：认证、遥测、模型调用全部依赖 AWS 服务

---

## 二十四、改进空间（与 Kiro CLI 对比，我的潜在进化方向）

相比 Kiro CLI，我可以在以下方面进化：

1. ✅ **工具扩展**：增加 `grep`、`ls`、`代码搜索` 等文件系统工具
2. ✅ **知识库**：引入检索增强生成（RAG）能力
3. ✅ **对话记忆**：实现更好的长对话管理和压缩
4. ✅ **权限模型**：在敏感操作前加入确认机制
5. ✅ **自定义 Agent**：允许用户通过配置文件定制行为
6. ✅ **子任务分解**：实现 TODO 列表式任务规划
7. ✅ **代码理解**：集成 Tree-sitter 或 LSP 能力
8. ✅ **MCP 协议**：支持第三方工具扩展标准

---

> 本报告基于对 kiro-cli 三个二进制文件（ELF 64-bit）的 **strings 分析**、**SQLite 数据库结构分析**、以及从二进制中提取的 **Rust crate 路径反推** 得出。
>
> 分析工具：`strings`、`file`、`sqlite3`
> 分析对象：`/root/.local/bin/kiro-cli`, `kiro-cli-chat`, `kiro-cli-term`
> 数据库：`/root/.local/share/kiro-cli/data.sqlite3`
>
> 关键源码仓库：`github.com/aws/amazon-q-developer-cli`
