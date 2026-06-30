# Oh My Pi (omp) — Code Wiki

> 版本: 16.2.9 | 许可证: MIT | 作者: Can Bölük | 主页: [omp.sh](https://omp.sh)

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [TypeScript 包详解](#3-typescript-包详解)
4. [Rust Crate 详解](#4-rust-crate-详解)
5. [依赖关系图](#5-依赖关系图)
6. [数据流与核心流程](#6-数据流与核心流程)
7. [项目运行方式](#7-项目运行方式)
8. [工具体系](#8-工具体系)
9. [扩展系统](#9-扩展系统)
10. [配置体系](#10-配置体系)

---

## 1. 项目概述

**Oh My Pi (omp)** 是一个终端优先的 AI 编码代理，由 [Pi](https://github.com/badlogic/pi-mono) (Mario Zechner) 分叉而来，并在此基础上扩展了完整的编码工作流。项目采用 TypeScript + Rust 混合架构，TypeScript 负责应用逻辑、Agent 运行时和 TUI 渲染，Rust 负责性能关键的原生操作（搜索、Shell 执行、AST 解析、语法高亮等）。

### 核心特性

- **40+ LLM 提供商** 支持，含 OAuth 自动发现与凭证轮换
- **32 个内置工具**，涵盖文件操作、代码执行、LSP/DAP 集成、浏览器自动化等
- **~55,000 行 Rust 原生代码**，实现零 fork-exec 的进程内搜索/Shell/AST
- **4 种运行模式**：交互式 TUI、单次打印、RPC (stdio)、ACP (编辑器集成)
- **子代理系统**：任务分片、工作树隔离、Schema 验证输出
- **Advisor 模型**：第二模型监督每轮对话
- **Hindsight 记忆**：跨会话持久化知识库
- **SnapCompact**：上下文压缩，减少 Token 消耗

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      omp CLI (coding-agent)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ Interactive│  │  Print   │  │   RPC    │  │     ACP        │  │
│  │   Mode    │  │   Mode   │  │   Mode   │  │   Mode         │  │
│  └─────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘  │
│        └──────────────┼─────────────┼────────────────┘           │
│                       ▼                                           │
│              ┌────────────────┐                                   │
│              │   SDK Layer    │  createAgentSession()             │
│              └───────┬────────┘                                   │
│                      ▼                                            │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    Agent Runtime (pi-agent-core)           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │   │
│  │  │Agent Loop│ │Compaction│ │Thinking  │ │Tool Execution│ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │   │
│  └──────────────────────┬────────────────────────────────────┘   │
│                         ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                   AI Client (pi-ai)                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │   │
│  │  │  Auth    │ │ Dialects │ │ Registry │ │   Streaming  │ │   │
│  │  │  Broker  │ │(anthropic│ │          │ │              │ │   │
│  │  │  Gateway │ │,openai,  │ │          │ │              │ │   │
│  │  │          │ │gemini…)  │ │          │ │              │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │   │
│  └──────────────────────┬────────────────────────────────────┘   │
│                         ▼                                         │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                  Model Catalog (pi-catalog)                │   │
│  │  models.json · Provider Descriptors · Identity/Classify   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │TUI (pi-tui) │ │Utils     │ │Hashline  │ │SnapCompact      │  │
│  │             │ │(pi-utils)│ │          │ │                 │  │
│  └─────────────┘ └──────────┘ └──────────┘ └─────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │              N-API Bridge (pi-natives JS → Rust)           │   │
│  └──────────────────────┬────────────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Rust Native Layer                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │pi-natives│ │ pi-shell │ │  pi-ast  │ │     pi-iso       │  │
│  │ (N-API)  │ │(bash/PTY)│ │(AST/摘要)│ │  (隔离后端)      │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌───────────────────────────────┐  │
│  │pi-uu-grep│ │pi-uutils │ │  Vendored: brush-core,        │  │
│  │(ripgrep) │ │  -ctx    │ │  brush-builtins, uu-*         │  │
│  └──────────┘ └──────────┘ └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. TypeScript 包详解

### 3.1 `packages/coding-agent` — 主 CLI 应用 (核心焦点)

**包名**: `@oh-my-pi/pi-coding-agent`

这是项目的主入口，整合了所有其他包，提供完整的终端 AI 编码代理体验。

#### 目录结构

```
src/
├── cli.ts              # CLI 入口点：参数解析、工作空间设置、模式分发
├── main.ts             # 主函数：初始化各子系统并启动运行
├── sdk.ts              # SDK 接口：createAgentSession() 等程序化 API
├── index.ts            # 包导出入口，统一导出所有公共 API
├── config.ts           # 配置加载与管理
├── system-prompt.ts    # 系统提示词构建（Handlebars 模板）
├── thinking.ts         # 思考模式/思维链管理
├── startup-splash.ts   # 启动画面渲染
├── workspace-tree.ts   # 工作空间文件树
├── telemetry-export.ts # 遥测数据导出
├── cursor.ts           # Cursor IDE 集成
├── priority.json       # 工具优先级配置
├── cli/                # CLI 子模块
│   └── args.ts         # 命令行参数定义与解析
├── modes/              # 运行模式
│   ├── interactive-mode.ts  # 交互式 TUI 模式
│   ├── print-mode.ts        # 单次打印模式 (omp -p)
│   └── rpc/                 # RPC 模式
│       └── rpc-mode.ts      # stdio JSON-RPC 服务
├── tools/              # 内置工具实现
│   ├── bash/           # Shell 执行工具
│   ├── read/           # 文件/URL/内部协议读取
│   ├── write/          # 文件写入
│   ├── edit/           # Hashline 编辑
│   ├── search/         # 正则搜索
│   ├── ast-edit/       # AST 结构化重写
│   ├── ast-grep/       # AST 模式匹配
│   ├── lsp/            # LSP 集成
│   ├── debug/          # DAP 调试器
│   ├── task/           # 子代理任务
│   ├── browser/        # 浏览器自动化
│   ├── web-search/     # Web 搜索
│   ├── github/         # GitHub CLI
│   ├── eval/           # Python/JS 代码执行
│   ├── memory/         # Hindsight 记忆
│   └── ...             # 其他 30+ 工具
├── session/            # 会话管理
│   ├── agent-session.ts     # Agent 会话核心
│   ├── session-manager.ts   # 会话生命周期管理
│   └── session-storage.ts   # 会话持久化
├── extensions/         # 扩展系统
│   ├── extension-loader.ts  # 扩展加载器
│   └── extensions-runner.ts # 扩展运行时
├── mcp/                # Model Context Protocol
│   ├── mcp-manager.ts       # MCP 服务器管理
│   └── mcp-connection.ts    # MCP 连接管理
├── config/             # 配置管理
│   ├── settings-manager.ts  # 设置管理
│   └── model-registry.ts    # 模型注册表
├── capabilities/       # 能力与规则
│   └── rulebook.ts          # 规则匹配管道
├── tui/                # TUI 组件（本地化）
│   ├── input-controller.ts  # 输入控制
│   ├── event-controller.ts  # 事件控制
│   └── tool-execution.ts    # 工具执行渲染
└── acp/                # Agent Client Protocol
    └── acp-server.ts        # ACP 服务端
```

#### 关键导出 (index.ts)

| 导出 | 类型 | 说明 |
|------|------|------|
| `ModelRegistry` | 类 | 模型注册与发现 |
| `SessionManager` | 类 | 会话生命周期管理 |
| `createAgentSession` | 函数 | 创建 Agent 会话的工厂函数 |
| `discoverAuthStorage` | 函数 | 自动发现认证存储 |
| `AgentSession` | 类 | Agent 会话实例 |
| `InteractiveMode` | 类 | 交互模式运行器 |

#### 四种运行模式

| 模式 | 入口 | 用途 |
|------|------|------|
| **Interactive** | `InteractiveMode` | 终端 TUI 交互，默认模式 |
| **Print** | `omp -p "prompt"` | 单次提示-回答，非交互 |
| **RPC** | `omp --mode rpc` | stdio NDJSON 协议，供外部程序驱动 |
| **ACP** | `omp acp` | Agent Client Protocol，编辑器集成 (Zed 等) |

---

### 3.2 `packages/ai` — 多提供商 LLM 客户端

**包名**: `@oh-my-pi/pi-ai`

统一的 LLM API 层，处理多提供商的认证、流式传输、方言适配和重试逻辑。

#### 目录结构

```
src/
├── index.ts            # 主入口，统一导出
├── auth-broker.ts      # 认证代理：OAuth/token 管理
├── auth-gateway.ts     # 认证网关：请求拦截注入凭证
├── auth-storage.ts     # 认证持久化存储
├── dialect/            # 提供商方言适配器
│   ├── anthropic/      # Anthropic Messages API
│   ├── openai/         # OpenAI Completions/Responses/Codex
│   ├── google/         # Google Gemini / Vertex AI
│   ├── xai/            # xAI Grok
│   └── ...             # 其他提供商
├── registry.ts         # 提供商注册表
├── usage.ts            # Token 使用量统计
├── stream.ts           # 流式响应处理
└── errors.ts           # 错误分类与处理
```

#### 关键类与函数

| 名称 | 说明 |
|------|------|
| `AuthBroker` | 管理 OAuth 流程和 Token 刷新 |
| `AuthGateway` | 拦截 LLM 请求，自动注入认证头 |
| `ApiRegistry` | 注册和查找各提供商的 API 实例 |
| `registerBuiltinProviders()` | 注册所有内置提供商 |
| 各 `dialect` 模块 | 将统一消息格式转为各提供商的 API 格式 |

#### 支持的提供商

Anthropic · OpenAI · OpenAI Codex · Google Gemini · xAI · Mistral · Groq · Cerebras · Fireworks · Together · Hugging Face · NVIDIA · OpenRouter · Cursor · GitHub Copilot · GitLab Duo · Kimi · MiniMax · Alibaba · Qwen · Z.AI/GLM · Xiaomi MiMo · Ollama · LM Studio · llama.cpp · vLLM · LiteLLM 等 40+ 提供商。

---

### 3.3 `packages/agent` — Agent 运行时

**包名**: `@oh-my-pi/pi-agent-core`

通用的 Agent 运行时，处理工具调用循环、上下文管理、压缩和遥测。

#### 目录结构

```
src/
├── index.ts            # 导出入口
├── agent.ts            # Agent 类：核心状态机
├── agent-loop.ts       # Agent 循环：消息→工具调用→结果→消息
├── compaction.ts       # 上下文压缩：摘要过长对话
├── thinking.ts         # 思维链/思考模式管理
├── proxy.ts            # 请求代理与重试
├── replay-policy.ts    # 工具结果重放策略
├── run-collector.ts    # 运行结果收集
├── telemetry.ts        # OpenTelemetry 遥测
├── tokenizer.ts        # Token 计数
├── types.ts            # 核心类型定义
└── append-only-context.ts  # 追加式上下文模式
```

#### 关键类与函数

| 名称 | 说明 |
|------|------|
| `Agent` | Agent 核心类，管理状态和工具调用 |
| `AgentLoop` | 主循环：发送消息 → 解析工具调用 → 执行 → 回传结果 |
| `compact()` | 上下文压缩，生成摘要替代过长对话 |
| `ThinkingManager` | 管理模型的思维链输出 |

---

### 3.4 `packages/catalog` — 模型目录

**包名**: `@oh-my-pi/pi-catalog`

集中管理模型数据库、提供商描述符和模型身份分类。

#### 目录结构

```
src/
├── index.ts                # 导出入口
├── models.json             # 生成的模型数据库（禁止手动编辑）
├── discovery.ts            # 模型与提供商自动发现
├── identity/               # 模型身份与分类
│   ├── classify.ts         # 模型家族/版本解析
│   └── equivalence.ts      # 模型等价性判断
├── provider-models/        # 提供商模型描述符
│   ├── descriptors.ts      # CATALOG_PROVIDERS 表
│   └── openai-compat.ts    # OpenAI 兼容提供商解析
├── model-thinking.ts       # 思维元数据与策略
├── model-manager.ts        # 模型管理器
└── model-cache.ts          # 模型缓存
```

#### 关键概念

- **models.json**: 由 `bun run gen:models` 从 `generate-models.ts` 生成，包含所有已知模型的元数据
- **Discovery**: 自动检测本地/云端可用的模型
- **Identity**: 模型分类 (GPT-4o, Claude Sonnet 4, Gemini 3 Flash 等) 和版本解析
- **描述符**: 每个提供商的默认模型、发现工厂、定价信息

> **重要**: 禁止直接编辑 `models.json`，应修改 `provider-models/` 下的源文件后重新生成。

---

### 3.5 `packages/tui` — 终端 UI 库

**包名**: `@oh-my-pi/pi-tui`

基于差异化渲染的高性能终端 UI 框架。

#### 核心组件

| 组件 | 说明 |
|------|------|
| `Box` | 基础容器组件 |
| `Editor` | 文本编辑器 |
| `Input` | 输入框 |
| `Markdown` | Markdown 渲染 |
| `Selector` | 选项选择器 |
| `Spinner` | 加载动画 |

#### 关键能力

- 差异化渲染：只更新变化的屏幕区域
- ANSI 感知的文本宽度计算与截断
- 键盘/鼠标事件处理
- Kitty 键盘协议 + xterm 回退
- 颜色主题系统

---

### 3.6 `packages/utils` — 共享工具

**包名**: `@oh-my-pi/pi-utils`

跨包共享的基础工具函数库。

#### 核心模块

| 模块 | 说明 |
|------|------|
| `logger` | Winston 日志器，输出到 `~/.omp/logs/omp.YYYY-MM-DD.log` |
| `env` | 环境检测、Worker 入口管理 |
| `paths` | 路径操作工具 |
| `stream` | 流读取辅助 (readStream, readLines) |
| `isEnoent` | ENOENT 错误判断 |
| `workerHostEntry()` | Worker 宿主入口解析 |
| `$which()` | 二进制查找 |

---

### 3.7 `packages/natives` — N-API 绑定桥

**包名**: `@oh-my-pi/pi-natives`

TypeScript 侧的 N-API 绑定层，将 Rust 原生能力暴露给 JS 运行时。

#### 导出模块

| 模块 | 说明 |
|------|------|
| `grep` | 进程内正则搜索 (ripgrep) |
| `glob` | 文件发现与 glob 匹配 |
| `shell` | 嵌入式 Bash 执行 |
| `highlight` | 语法高亮 |
| `text` | ANSI 文本宽度/截断/换行 |
| `summary` | Tree-sitter 源码摘要 |
| `ast` | AST 模式匹配 |
| `pty` | 原生 PTY 分配 |
| `clipboard` | 剪贴板读写 |
| `tokens` | BPE Token 计数 |
| `sixel` | 终端图像渲染 |
| `fsCache` | 文件缓存系统 |
| `workspace` | 工作空间遍历 |
| `appearance` | 深色/浅色模式检测 |
| `power` | macOS 防睡眠断言 |
| `prof` | 性能分析器 |
| `ps` | 进程树操作 |
| `iso` | 工作空间隔离 |
| `html` | HTML → Markdown |
| `keys` | 键盘协议解析 |

---

### 3.8 `packages/hashline` — Hashline 编辑协议

**包名**: `@oh-my-pi/hashline`

基于内容哈希锚点的行级补丁语言和执行器，是 `edit` 工具的核心。

#### 核心概念

- **锚点 (Anchor)**: 用内容哈希标记目标行，而非行号
- **补丁 (Patch)**: 描述增/删/改操作的指令序列
- **过期检测**: 文件变更导致锚点偏移时自动拒绝补丁

---

### 3.9 `packages/mnemopi` — 记忆引擎

**包名**: `@oh-my-pi/pi-mnemopi`

本地 SQLite 驱动的 Agent 记忆系统，支持 `retain`/`recall`/`reflect` 三级操作。

#### 核心模块

- **core/**: 记忆存储与检索核心
- **dr/**: 距离度量与嵌入向量
- **util/**: 工具函数

---

### 3.10 `packages/snapcompact` — 上下文压缩

**包名**: `@oh-my-pi/snapcompact`

位图帧上下文压缩包，减少长对话的 Token 消耗。包含 SQuAD 评估套件和多种压缩策略实验。

---

### 3.11 `packages/collab-web` — 协作 Web 客户端

**包名**: `@oh-my-pi/collab-web`

浏览器端的实时协作查看器，支持通过链接或 QR 码共享会话。

---

### 3.12 `packages/wire` — 协议类型

**包名**: `@oh-my-pi/pi-wire`

协作会话的共享协议类型定义和 Relay 常量。

---

### 3.13 `packages/stats` — 可观测性仪表板

**包名**: `@oh-my-pi/omp-stats`

本地 AI 使用统计仪表板 (`omp stats`)，提供用量可视化。

---

### 3.14 `packages/swarm-extension` — Swarm 扩展

**包名**: `@oh-my-pi/swarm-extension`

Swarm 编排扩展包，提供多 Agent 协作编排能力。

---

## 4. Rust Crate 详解

### 4.1 `pi-natives` — 核心 N-API 原生插件

**类型**: `cdylib` (N-API addon)

聚合所有 Rust 原生能力，通过 N-API 暴露给 JavaScript 运行时。

#### 模块与 ~LoC

| 模块 | 功能 | 底层库 | ~LoC |
|------|------|--------|-----:|
| `shell` | 嵌入式 Bash、持久会话、超时/中止 | brush-shell (vendored) | 3,700 |
| `grep` | 正则搜索、并行/顺序、glob 过滤 | grep-regex · grep-searcher | 1,900 |
| `keys` | Kitty 键盘协议 + xterm 回退 | phf | 1,490 |
| `text` | ANSI 感知宽度/截断/换行 | unicode-width · segmentation | 1,450 |
| `summary` | Tree-sitter 结构化源码摘要 | tree-sitter · ast-grep-core | 1,040 |
| `ast` | ast-grep 模式匹配与重写 | ast-grep-core | 1,000 |
| `fs_cache` | mtime 键控文件缓存 | in-tree | 840 |
| `highlight` | 语法高亮 (11 语义类别) | syntect | 470 |
| `pty` | 原生 PTY 分配 | portable-pty | 455 |
| `glob` | 文件发现、glob、gitignore | ignore · globset | 410 |
| `workspace` | 工作空间遍历 + AGENTS.md 发现 | ignore | 385 |
| `appearance` | 深浅模式检测 (macOS FFI) | core-foundation | 270 |
| `power` | macOS 防睡眠断言 | IOKit FFI | 270 |
| `task` | libuv 线程池阻塞任务 | tokio · napi | 260 |
| `fd` | 文件系统遍历器 (find 替代) | ignore | 250 |
| `iso` | 工作空间隔离 | pi-iso (PAL) | 245 |
| `prof` | 循环缓冲区性能分析器 | inferno | 240 |
| `ps` | 跨平台进程树操作 | libc · libproc | 195 |
| `clipboard` | 剪贴板读写 | arboard | 80 |
| `tokens` | O200k/Cl100k BPE Token 计数 | tiktoken-rs | 65 |
| `sixel` | 终端图像渲染 | icy_sixel · image | 55 |
| `html` | HTML → Markdown | html-to-markdown-rs | 50 |

#### 关键依赖

```toml
pi-ast, pi-shell, pi-iso          # 内部 crate
napi, napi-derive, napi-build      # N-API 绑定
tiktoken-rs                         # Token 计数
syntect                             # 语法高亮
tree-sitter, ast-grep-core          # AST 操作
grep-regex, grep-searcher, ignore   # 搜索
brush-core, brush-builtins          # 嵌入式 Shell
portable-pty                        # PTY
image, icy_sixel                    # 图像处理
arboard                             # 剪贴板
inferno                             # 火焰图
```

---

### 4.2 `pi-shell` — 嵌入式 Shell

**功能**: 在进程内运行 Bash 命令，无需 fork-exec。

#### 核心能力

- 包装 `brush-core` (vendored Bash 实现)
- 持久会话：Shell 状态跨调用保持
- 内置 Minimizer：智能压缩命令输出（50+ 预定义规则）
- Vendored uutils: `cat`, `find`, `head`, `ls`, `mkdir`, `mv`, `rm`, `sort`, `tail`, `uniq`, `wc`
- 取消支持：超时和主动中止
- 跨平台：macOS、Linux、Windows

#### 导出

```rust
pub use shell::Shell;       // 核心 Shell 类型
pub mod exec;               // 执行选项
pub mod run;                 // 运行 API
```

---

### 4.3 `pi-ast` — AST 工具

**功能**: 基于 tree-sitter 的代码摘要和 AST 操作。

#### 支持 50+ 语言

tree-sitter-bash, -c, -cpp, -css, -go, -html, -java, -javascript, -json, -kotlin, -lua, -python, -ruby, -rust, -scala, -swift, -typescript, -yaml, -zig 等。

#### 导出

```rust
pub mod block;       // 代码块识别
pub mod language;    // 语言解析器注册
pub mod ops;         // AST 操作
pub mod summary;     // 结构化源码摘要
```

---

### 4.4 `pi-iso` — 工作空间隔离

**功能**: 为子代理任务提供文件系统隔离，支持多种后端。

#### 隔离后端

| 后端 | 平台 | 方式 |
|------|------|------|
| APFS | macOS | APFS 克隆 |
| btrfs | Linux | btrfs reflink |
| ZFS | Linux | ZFS 克隆 |
| overlayfs | Linux | OverlayFS 挂载 |
| projfs | Windows | ProjFS 虚拟化 |
| rcopy | 全平台 | 递归文件复制 (回退) |

#### 导出

```rust
pub mod apfs, btrfs, diff, linux_reflink, overlayfs, projfs, rcopy, windows_block_clone, zfs;
```

---

### 4.5 `pi-uu-grep` — 进程内 grep

**功能**: 基于 ripgrep 库的 grep 实现，用作 Shell 内置命令。

```rust
pub mod rg;     // ripgrep 封装
pub fn run();   // grep 执行入口
```

---

### 4.6 `pi-uutils-ctx` — uutils 上下文

**功能**: 为在 Shell 内执行 uutils 工具提供 I/O 和工作目录上下文。

关键函数: `scope()`, `resolve()`, `stdout()`, `stdin()`, `stderr()`

---

## 5. 依赖关系图

```
                    ┌─────────────┐
                    │ coding-agent│
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      ┌─────────┐   ┌──────────┐   ┌───────────┐
      │  agent  │   │   ai     │   │catalog    │
      └────┬────┘   └────┬─────┘   └─────┬─────┘
           │              │               │
           ├──────────────┤               │
           ▼              ▼               ▼
      ┌─────────┐   ┌──────────┐   ┌───────────┐
      │  tui    │   │ natives  │   │  utils    │
      └────┬────┘   └────┬─────┘   └───────────┘
           │              │               ▲
           ▼              ▼               │
      ┌─────────┐   ┌──────────┐         │
      │ natives │   │ natives  │─────────┘
      │  (Rust) │   │  (JS)    │
      └────┬────┘   └──────────┘
           │
     ┌─────┼──────┬──────────┐
     ▼     ▼      ▼          ▼
  ┌─────┐┌─────┐┌─────┐ ┌──────┐
  │shell││ ast ││ iso │ │uu-grep│
  └──┬──┘└─────┘└─────┘ └──────┘
     │
     ▼
  ┌────────────────────────────────────┐
  │ Vendored: brush-core, brush-builtins│
  │ uu-cat, uu-find, uu-head, uu-ls…   │
  └────────────────────────────────────┘
```

### 包间依赖矩阵

| 包 → 依赖 | agent | ai | catalog | tui | utils | natives | hashline | mnemopi | snapcompact | wire | stats |
|-----------|:-----:|:--:|:-------:|:---:|:-----:|:-------:|:--------:|:-------:|:-----------:|:----:|:-----:|
| coding-agent | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | |
| agent | | ✓ | ✓ | | ✓ | ✓ | | | ✓ | ✓ | |
| ai | | | ✓ | | ✓ | | | | | ✓ | |
| tui | | | | | ✓ | ✓ | | | | | |
| catalog | | | | | ✓ | | | | | | |
| hashline | | | | | ✓ | ✓ | | | | | |
| mnemopi | | | | | ✓ | ✓ | | | | | |

---

## 6. 数据流与核心流程

### 6.1 交互式会话流程

```
用户输入 → InputController → AgentSession.prompt()
                              │
                              ▼
                         Agent Loop
                    ┌──────────────┐
                    │ 构建消息历史  │
                    │  + 系统提示   │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ AI Client    │
                    │ (pi-ai)      │
                    │ → 流式请求   │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ 解析响应      │
                    │ 文本/工具调用  │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
              ┌─────│ 工具调用？    │─────┐
              │     └──────────────┘     │
              否                          是
              │                          ▼
              ▼                   ┌──────────────┐
         渲染文本输出             │ 工具执行      │
         (TUI Markdown)          │ (bash/read/  │
                                 │  edit/…)     │
                                 └──────┬───────┘
                                        ▼
                                 ┌──────────────┐
                                 │ 回传结果      │
                                 │ → Agent Loop │
                                 └──────────────┘
                                        │
                                        ▼
                                   (继续循环)
```

### 6.2 工具调用路由

```
Tool Call (from LLM)
    │
    ├── bash → pi-natives.shell (Rust brush-shell)
    ├── read → pi-natives.fsCache + pi-natives.summary
    ├── edit → hashline (内容哈希补丁)
    ├── ast_edit → pi-natives.ast (ast-grep)
    ├── search → pi-natives.grep (ripgrep)
    ├── lsp → LSP Client (typescript-language-client)
    ├── debug → DAP Client (debug adapter protocol)
    ├── task → 子 AgentSession (工作树隔离)
    ├── browser → Puppeteer (Chromium)
    ├── web_search → 搜索提供商链 (18 后端)
    ├── eval → Python/JS Worker
    └── memory → mnemopi (SQLite)
```

### 6.3 认证流程

```
请求 → AuthGateway
          │
          ├── 检查缓存的凭证
          │     │
          │     ├── 有效 → 注入 Authorization 头 → 发送
          │     │
          │     └── 过期/缺失 → AuthBroker
          │                        │
          │                        ├── OAuth 流程 (浏览器)
          │                        ├── API Key (配置文件)
          │                        └── 编码计划 (订阅)
          │
          └── 429/401 响应 → 凭证轮换 → 重试
```

---

## 7. 项目运行方式

### 7.1 环境要求

- **Bun** ≥ 1.3.14 (包管理器 + 运行时)
- **Rust** (stable, edition 2024) — 用于编译原生模块
- **Node.js** — N-API 兼容 (通过 Bun 运行时)

### 7.2 从源码构建

```bash
# 完整安装（依赖 + Rust 原生模块）
bun setup

# 开发模式运行
bun dev

# 仅重新构建 Rust 原生模块
bun run build:native

# 非交互验证
bun dev -- --version
```

### 7.3 常用开发命令

| 命令 | 说明 |
|------|------|
| `bun setup` | 安装依赖 + 构建原生模块 + 链接 CLI |
| `bun dev` | 以开发模式启动 CLI |
| `bun run build` | 构建所有工作空间包 |
| `bun run build:native` | 仅构建 Rust N-API 模块 |
| `bun test` | 运行所有 TS + RS 测试 |
| `bun run test:ts` | 运行 TypeScript 测试 |
| `bun run test:rs` | 运行 Rust 测试 |
| `bun run check` | 类型检查 (TS + RS) |
| `bun run check:ts` | TypeScript 类型检查 |
| `bun run lint` | 代码检查 (TS + RS) |
| `bun run fmt` | 代码格式化 (TS + RS) |
| `bun run fix` | 自动修复 (TS + RS) |
| `bun run gen:models` | 重新生成 models.json |
| `bun run release` | 版本发布 |

### 7.4 CI 管道

```bash
bun run ci:check:full          # 完整类型检查
bun run ci:test:full           # 完整测试
bun run ci:test:smoke          # 冒烟测试 (--version --help --smoke-test)
bun run ci:test:install-methods # 安装方式测试
bun run ci:release:build-binaries  # 构建二进制
bun run ci:release:publish     # 发布
```

### 7.5 生产安装

```bash
# macOS / Linux
curl -fsSL https://omp.sh/install | sh

# Homebrew
brew install can1357/tap/omp

# Bun (推荐)
bun install -g @oh-my-pi/pi-coding-agent

# Windows (PowerShell)
irm https://omp.sh/install.ps1 | iex
```

---

## 8. 工具体系

omp 内置 32 个工具，分为以下类别：

### 文件与搜索

| 工具 | 功能 | 底层实现 |
|------|------|----------|
| `read` | 读取文件/目录/归档/SQLite/PDF/URL/内部协议 | pi-natives.fsCache + summary |
| `write` | 创建/覆盖文件 | Bun.write |
| `edit` | Hashline 补丁编辑 | hashline 包 |
| `ast_edit` | AST 结构化重写 (预览→确认) | pi-natives.ast |
| `ast_grep` | AST 模式查询 (50+ 语法) | pi-natives.ast |
| `search` | 正则搜索 | pi-natives.grep |
| `find` | Glob 路径查找 | pi-natives.glob |

### 运行时

| 工具 | 功能 | 底层实现 |
|------|------|----------|
| `bash` | 工作空间 Shell | pi-natives.shell |
| `eval` | Python/JS 代码执行 | Worker + 工具回调桥 |
| `ssh` | 远程命令 | SSH 协议 |

### 代码智能

| 工具 | 功能 | 底层实现 |
|------|------|----------|
| `lsp` | 诊断/导航/符号/重命名/代码操作 | LSP Client |
| `debug` | DAP 调试 (断点/步进/变量) | DAP Client |

### 协调

| 工具 | 功能 |
|------|------|
| `task` | 并行子代理 (工作树隔离) |
| `irc` | Agent 间通信 |
| `todo` | 会话 Todo 列表 |
| `job` | 后台任务管理 |
| `ask` | 结构化用户提问 |

### 外部集成

| 工具 | 功能 |
|------|------|
| `browser` | Puppeteer 浏览器自动化 |
| `web_search` | 18 提供商搜索链 |
| `github` | GitHub CLI 操作 |

### 记忆与状态

| 工具 | 功能 |
|------|------|
| `checkpoint` | 标记对话状态 |
| `rewind` | 裁剪探索性上下文 |
| `retain` | 持久化事实 |
| `recall` | 搜索记忆库 |
| `reflect` | 综合记忆回答 |

### 内部协议

| 协议 | 格式 | 说明 |
|------|------|------|
| `pr://` | `read pr://owner/repo/number` | 读取 PR 内容 |
| `issue://` | `read issue://owner/repo/number` | 读取 Issue |
| `agent://` | `read agent://id/findings.0.path` | 读取子代理输出 |
| `skill://` | `read skill://name` | 读取 Skill 定义 |
| `rule://` | `read rule://name` | 读取规则 |
| `conflict://` | `write conflict://1 @theirs` | 解决合并冲突 |

---

## 9. 扩展系统

### 扩展发现

omp 首次运行时自动继承磁盘上的配置：

- `.claude/` — Claude 规则
- `.cursor/` — Cursor MDC
- `.windsurf/` — Windsurf 规则
- `.gemini/` — Gemini 规则
- `.codex/` — Codex AGENTS.md
- `.cline/` — Cline 规则
- `.github/copilot/` — Copilot applyTo
- `.vscode/` — VS Code 设置

### 扩展能力

扩展是 TypeScript 模块，可注册：

- 自定义工具
- Slash 命令
- 快捷键绑定
- TUI 组件
- MCP 服务器

### MCP (Model Context Protocol)

支持 stdio/SSE 传输的 MCP 服务器，自动发现和生命周期管理。

---

## 10. 配置体系

### 配置文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| 全局设置 | `~/.omp/agent/settings.yml` | 全局 Agent 配置 |
| 项目设置 | `.omp/settings.yml` | 项目级配置 |
| 模型配置 | `~/.omp/agent/models.yml` | 自定义提供商/模型 |
| 认证存储 | `~/.omp/auth/` | OAuth Token 和 API Key |
| 日志 | `~/.omp/logs/omp.YYYY-MM-DD.log` | 运行日志 |
| 会话 | `~/.omp/sessions/` | 持久化会话数据 |
| 记忆 | `~/.omp/mnemopi/` | Hindsight 记忆库 |

### 角色系统

| 角色 | 用途 | CLI 标志 |
|------|------|----------|
| `default` | 正常对话 | (默认) |
| `smol` | 低成本子代理 | `--smol` |
| `slow` | 深度推理 | `--slow` |
| `plan` | 规划模式 | `--plan` |
| `commit` | 提交消息 | `--commit` |

### 关键配置项

- `enabledModels` / `disabledProviders`: 模型白/黑名单
- `retry.fallbackChains`: 每角色的回退链
- `path:` 前缀: 路径范围限定的模型配置
- `tools.discoveryMode`: 工具自动发现模式

---

## 附录: 版本发布流程

1. 确认所有变更已记录在各包的 `[Unreleased]` 段
2. 执行 `bun run release`
3. 脚本自动完成：版本号升级、CHANGELOG 定稿、Git 提交/标签、npm 发布、新 `[Unreleased]` 段创建
