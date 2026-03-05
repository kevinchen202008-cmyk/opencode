# OpenCode 系统架构文档

## 1. 整体架构 - Monorepo 结构

OpenCode 采用 **Turborepo** + **Bun** 的 Monorepo 结构，使用 workspace 管理多个包。

### 核心包列表

| 包名         | 路径                | 作用                                                            |
| ------------ | ------------------- | --------------------------------------------------------------- |
| **opencode** | `packages/opencode` | 核心 CLI 和引擎，包含 agent、provider、session、tool 等核心模块 |
| **function** | `packages/function` | Cloudflare Workers 云函数，处理同步/分享功能                    |
| **sdk**      | `packages/sdk`      | SDK 包，生成客户端 SDK (JS/TS)                                  |
| **ui**       | `packages/ui`       | UI 组件库                                                       |
| **app**      | `packages/app`      | Web 应用 (SolidJS)                                              |
| **desktop**  | `packages/desktop`  | 桌面应用 (Tauri)                                                |
| **web**      | `packages/web`      | Web 相关组件                                                    |
| **console**  | `packages/console`  | 控制台相关                                                      |
| **plugin**   | `packages/plugin`   | 插件 SDK                                                        |
| **script**   | `packages/script`   | 脚本工具                                                        |
| **util**     | `packages/util`     | 通用工具库                                                      |

---

## 2. 核心模块关系

### 2.1 核心文件结构

```
packages/opencode/src/
├── agent/          # Agent 定义和配置
├── provider/       # AI Provider 管理
├── session/        # 会话管理
├── tool/           # 工具系统
├── lsp/            # LSP (Language Server Protocol) 客户端
├── plugin/         # 插件系统
├── mcp/            # MCP (Model Context Protocol)
├── storage/        # 数据库存储
├── auth/           # 认证系统
├── config/         # 配置管理
├── server/         # HTTP API 服务器
└── ...
```

### 2.2 核心模块关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        User Input                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      Server (HTTP)                          │
│              packages/opencode/src/server/                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Session (会话管理)                              │
│         packages/opencode/src/session/index.ts             │
│  - 创建会话                                                │
│  - 消息管理                                                │
│  - 状态追踪                                                │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   Agent     │   │   Provider  │   │    Tool     │
│  (agent.ts) │   │ (provider.ts)│   │  (tool.ts)  │
├─────────────┤   ├─────────────┤   ├─────────────┤
│ - build     │   │ - AI SDK    │   │ - read      │
│ - plan      │   │ - 20+厂商   │   │ - write     │
│ - explore   │   │ - OAuth/API │   │ - edit      │
│ - general   │   │ - 认证      │   │ - bash     │
│ - 自定义    │   │             │   │ - grep     │
└─────────────┘   └─────────────┘   │ - glob     │
                          │         │ - lsp     │
                          │         └─────────────┘
                          │
           ┌──────────────┴──────────────┐
           │         LSP Layer            │
           ├─────────────────────────────┤
           │  ┌─────────────────────┐    │
           │  │  LSP Client         │    │  opencode-cli.exe
           │  │ (本地 JSON-RPC)     │────┼── stdio ──▶ LSP Server
           │  └─────────────────────┘    │  (pyright, clangd...)
           │                              │
           │  ┌─────────────────────┐    │
           │  │  LSP Server         │    │  本地子进程
           │  │ (spawn 本地进程)    │    │
           │  └─────────────────────┘    │
           └─────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   LLM (流式调用)      │
              │ packages/opencode/src │
              │     session/llm.ts    │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │     AI Model          │
              │ (OpenAI/Anthropic/...) │
              └───────────────────────┘
```

---

## 3. 数据流 - 用户输入到 AI 响应

### 3.1 处理流程

```
User Input (命令行/TUI)
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ Server Route (routes/session.ts)                             │
│ - POST /session/:id/message                                  │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│ Session Processor (session/processor.ts)                     │
│ 1. 创建 Assistant Message                                    │
│ 2. 调用 LLM.stream()                                         │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│ LLM.stream() (session/llm.ts)                                │
│ 1. 获取 Provider 和 Model                                    │
│ 2. 构建 System Prompt                                        │
│ 3. 解析 Tools                                                │
│ 4. 调用 ai-sdk streamText()                                 │
└─────────────────────────┬────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    Text Stream      Tool Calls      Reasoning
          │               │               │
          │               ▼               │
          │       ┌───────────────┐       │
          │       │ Tool Registry │       │
          │       │ (tool/*.ts)   │       │
          │       └───────┬───────┘       │
          │               │               │
          └───────┬───────┴───────────────┘
                  │
                  ▼
        ┌─────────────────┐
        │ Response to    │
        │ User            │
        └─────────────────┘
```

---

## 4. Agent 系统

### 4.1 内置 Agent

| Agent          | 模式     | 用途                 |
| -------------- | -------- | -------------------- |
| **build**      | primary  | 默认 agent，执行工具 |
| **plan**       | primary  | 计划模式，禁止编辑   |
| **explore**    | subagent | 快速探索代码库       |
| **general**    | subagent | 通用研究任务         |
| **compaction** | primary  | 压缩任务 (隐藏)      |
| **title**      | primary  | 标题生成 (隐藏)      |
| **summary**    | primary  | 摘要生成 (隐藏)      |

### 4.2 Agent 配置

可通过 `opencode.json` 扩展或覆盖内置 agent:

```json
{
  "agent": {
    "my-agent": {
      "description": "Custom agent",
      "mode": "subagent",
      "model": "provider/model",
      "permission": { ... }
    }
  }
}
```

---

## 5. Provider 系统

### 5.1 支持的 AI 厂商

| Provider       | SDK                         | 说明         |
| -------------- | --------------------------- | ------------ |
| OpenAI         | @ai-sdk/openai              | GPT 系列     |
| Anthropic      | @ai-sdk/anthropic           | Claude 系列  |
| Google         | @ai-sdk/google              | Gemini 系列  |
| Azure          | @ai-sdk/azure               | Azure OpenAI |
| OpenRouter     | @openrouter/ai-sdk-provider | 聚合多模型   |
| 智谱 (ZhipuAI) | @ai-sdk/openai-compatible   | GLM 系列     |
| GitHub Copilot | @ai-sdk/github-copilot      | Copilot      |
| ...            | ...                         | 20+ 厂商     |

### 5.2 Provider 认证类型

```typescript
Auth.Info = z.discriminatedUnion("type", [
  Oauth: { type: "oauth", refresh, access, expires },
  Api: { type: "api", key },
  WellKnown: { type: "wellknown", key, token }
])
```

---

## 6. Tool 系统

### 6.1 内置工具

| 工具            | 功能               |
| --------------- | ------------------ |
| **read**        | 读取文件内容       |
| **write**       | 写入文件           |
| **edit**        | 编辑文件           |
| **glob**        | 文件搜索           |
| **grep**        | 内容搜索           |
| **bash**        | 执行 shell 命令    |
| **lsp**         | LSP 语言服务器功能 |
| **web-search**  | 网络搜索           |
| **file-search** | 文件搜索           |

### 6.2 工具定义

```typescript
Tool.define("read", async () => ({
  description: "Read file contents",
  parameters: z.object({ path: z.string() }),
  async execute(args, ctx) {
    return { title, metadata, output }
  },
}))
```

---

## 7. LSP (Language Server Protocol)

### 7.1 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenCode 架构中的 LSP                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   opencode-cli.exe                       │    │
│  │  ┌─────────────────────────────────────────────────────┐ │    │
│  │  │  LSP Client (lsp/client.ts)                        │ │    │
│  │  │  - 管理 LSP 服务器生命周期                           │ │    │
│  │  │  - JSON-RPC 通信 (vscode-jsonrpc)                  │ │    │
│  │  │  - 诊断结果收集、符号搜索、跳转定义...              │ │    │
│  │  └────────────────────────┬────────────────────────────┘ │    │
│  │                           │ stdio                         │    │
│  │  ┌────────────────────────▼────────────────────────────┐ │    │
│  │  │  LSP Server (本地子进程，由 lsp/server.ts spawn)     │ │    │
│  │  │                                                     │ │    │
│  │  │  • pyright      → Python                            │ │    │
│  │  │  • typescript   → TypeScript/JavaScript            │ │    │
│  │  │  • clangd       → C/C++                            │ │    │
│  │  │  • gopls        → Go                               │ │    │
│  │  │  • rust-analyzer → Rust                            │ │    │
│  │  │  • jdtls        → Java                            │ │    │
│  │  │  • ...         → 更多语言...                        │ │    │
│  │  └─────────────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 核心文件

```
packages/opencode/src/lsp/
├── index.ts       # 主入口，管理 LSP 客户端
├── server.ts      # LSP 服务器定义和自动安装 (spawn)
├── client.ts      # LSP 客户端实现 (JSON-RPC over stdio)
└── language.ts    # 语言扩展名映射
```

### 7.3 LSP 通信流程

```
1. 根据文件扩展名匹配 LSP Server
       │
       ▼
2. 查找项目根目录 (package.json, go.mod, Cargo.toml...)
       │
       ▼
3. spawn LSP Server 子进程
       │
       ▼
4. LSP Client 通过 stdio 与 Server 通信
       │
       ▼
5. LSP 功能:
   ┌─────────────┬─────────────┬─────────────┬─────────────┐
   │   hover    │ definition  │ references  │ diagnostics │
   └─────────────┴─────────────┴─────────────┴─────────────┘
```

### 7.4 支持的语言服务器

| Server ID          | 语言                  | 自动安装        |
| ------------------ | --------------------- | --------------- |
| `typescript`       | TypeScript/JavaScript | ✅ (via Bun)    |
| `deno`             | Deno                  | ❌              |
| `pyright`          | Python                | ✅ (via Bun)    |
| `ty`               | Python (实验)         | ❌              |
| `biome`            | JS/TS/JSON/CSS        | ✅ (via Bun)    |
| `oxlint`           | JS/TS (oxc)           | ✅ (via Bun)    |
| `eslint`           | ESLint                | ✅ (构建)       |
| `gopls`            | Go                    | ✅ (go install) |
| `rust-analyzer`    | Rust                  | ❌              |
| `clangd`           | C/C++                 | ✅ (GitHub)     |
| `jdtls`            | Java                  | ✅ (Eclipse)    |
| `kotlin-ls`        | Kotlin                | ✅ (GitHub)     |
| `elixir-ls`        | Elixir                | ✅ (编译)       |
| `svelte`           | Svelte                | ✅ (via Bun)    |
| `astro`            | Astro                 | ✅ (via Bun)    |
| `vue`              | Vue                   | ✅ (via Bun)    |
| `ruby-lsp`         | Ruby                  | ✅ (gem)        |
| `zls`              | Zig                   | ✅ (GitHub)     |
| `lua-ls`           | Lua                   | ✅ (GitHub)     |
| `csharp`           | C#                    | ✅ (dotnet)     |
| `fsharp`           | F#                    | ✅ (dotnet)     |
| `sourcekit-lsp`    | Swift                 | ❌              |
| `php intelephense` | PHP                   | ✅ (via Bun)    |
| `dart`             | Dart                  | ❌              |
| `ocaml-lsp`        | OCaml                 | ❌              |
| `bash`             | Bash                  | ✅ (via Bun)    |
| `terraform`        | Terraform             | ✅ (HashiCorp)  |
| `yaml-ls`          | YAML                  | ✅ (via Bun)    |
| `prisma`           | Prisma                | ❌              |

### 7.5 工作流程

```
1. 项目初始化
   │
   ▼
2. 根据文件扩展名匹配 LSP Server
   │
   ▼
3. 查找项目根目录 (package.json, go.mod, Cargo.toml...)
   │
   ▼
4. Spawn LSP Server 进程
   │
   ▼
5. LSP Client 连接 (JSON-RPC over stdio)
   │
   ▼
6. 提供 LSP 功能:
   - hover (悬停提示)
   - definition (跳转定义)
   - references (查找引用)
   - diagnostics (诊断错误)
   - symbol (符号搜索)
   - 等等...
```

### 7.6 LSP 工具

作为 Agent 工具暴露给 AI:

```
lsp({
  operation: "goToDefinition" | "findReferences" | "hover" |
             "documentSymbol" | "workspaceSymbol" | "goToImplementation" |
             "prepareCallHierarchy" | "incomingCalls" | "outgoingCalls",
  filePath: string,
  line: number,
  character: number
})
```

### 7.7 配置 (opencode.json)

```json
{
  "lsp": {
    "pyright": {
      "disabled": false,
      "command": ["pyright-langserver", "--stdio"],
      "env": { "PYTHONPATH": "/path" }
    }
  }
}
```

### 7.8 核心实现

- **LSP Client**: 使用 `vscode-jsonrpc` 通过 stdio 与本地 LSP Server 通信
- **LSP Server**: 使用 `child_process.spawn` 启动本地子进程
- **按需启动**: 只在需要时启动对应文件的 LSP 服务器
- **缓存复用**: 同一项目的 LSP 客户端会被复用
- **自动安装**: 首次使用时自动下载并安装 LSP 服务器二进制

### 7.9 网络需求

| 阶段                                    | 网络需求          |
| --------------------------------------- | ----------------- |
| LSP 服务器下载                          | ✅ 需要 (仅首次)  |
| LSP 运行时                              | ❌ 纯本地 (stdio) |
| LSP 功能 (hover/definition/diagnostics) | ❌ 纯本地         |

---

## 8. 插件系统

### 8.1 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Plugin System                            │
│              packages/opencode/src/plugin/                 │
├─────────────────────────────────────────────────────────────┤
│ Hooks (接口定义):                                          │
│ - experimental.chat.system.transform                       │
│ - chat.params                                              │
│ - chat.headers                                             │
│ - event                                                    │
│ - auth                                                     │
│ - tool                                                     │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 插件类型

**内置插件:**

- `opencode-anthropic-auth` - Anthropic 认证
- `CodexAuthPlugin` - OpenAI Codex 认证
- `CopilotAuthPlugin` - GitHub Copilot 认证
- `GitlabAuthPlugin` - GitLab 认证

**外部插件 (npm):**

- `opencode-antigravity-auth` - Google Antigravity OAuth
- `@tarquinen/opencode-dcp` - DCP 集成

### 8.3 插件配置

```json
{
  "plugin": ["opencode-antigravity-auth@latest", "@tarquinen/opencode-dcp@latest"]
}
```

---

## 9. 存储 - 数据库结构

### 9.1 技术栈

- **数据库**: SQLite (Drizzle ORM)
- **位置**: 每个项目独立的 SQLite 数据库
- **迁移**: Drizzle Kit

### 9.2 核心表结构

**session**: 会话表

- id, project_id, parent_id, slug, directory, title, version
- share_url, summary_additions/deletions/files
- permission (JSON - PermissionNext.Ruleset)
- time_created/updated/compacting/archived

**message**: 消息表

- id, session_id, time_created
- data (JSON - MessageV2.Info)

**part**: 消息内容片段表

- id, message_id, session_id, time_created
- data (JSON - text/tool/reasoning/attachment)

**todo**: 待办事项表

- session_id (复合主键)
- content, status, priority, position
- time_created/updated

**permission**: 权限表

- project_id (主键)
- time_created/updated
- data (JSON - PermissionNext.Ruleset)

---

## 10. GitHub CI/CD 集成

### 10.1 概述

OpenCode 可以作为 GitHub Agent 运行在 GitHub Actions 中，实现自动化代码审查、Issue 处理和 PR 自动化。

### 10.2 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub CI/CD 架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  Issue      │    │  PR Comment │    │  Workflow   │        │
│  │  Comment    │    │  Review     │    │  Dispatch  │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              GitHub Actions Workflow                     │    │
│  │              (.github/workflows/opencode.yml)            │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           opencode-agent[bot] (GitHub App)              │    │
│  │           - 认证授权                                      │    │
│  │           - Issue/PR 操作                                 │    │
│  │           - 分支/PR 管理                                  │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              OpenCode Session                            │    │
│  │  - LLM 对话                                             │    │
│  │  - Tool 执行 (read/write/edit/bash/grep...)            │    │
│  │  - Git 操作 (checkout/branch/commit/push)              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.3 支持的触发事件

| 事件类型                      | 描述        | 用途                 |
| ----------------------------- | ----------- | -------------------- |
| `issue_comment`               | Issue 评论  | Issue 讨论、命令触发 |
| `pull_request_review_comment` | PR 代码评论 | PR 代码审查          |
| `issues`                      | Issue 事件  | Issue 创建/编辑      |
| `pull_request`                | PR 事件     | PR 自动化            |
| `schedule`                    | CRON 定时   | 定时任务自动化       |
| `workflow_dispatch`           | 手动触发    | 手动运行任务         |

### 10.4 使用方式

**1. 安装 GitHub App**

```bash
opencode github install
```

这将：

- 安装 OpenCode GitHub App (`opencode-agent[bot]`)
- 选择 Provider 和 Model
- 生成 `.github/workflows/opencode.yml` 文件

**2. 触发 Agent**

```
# Issue 中评论
/oc 修复这个 bug

/oc summarize  # 总结

/opencode review  # 审查代码

# PR 中评论
/oc 这个函数可以优化
```

**3. 手动触发**

```bash
# workflow_dispatch
gh workflow run opencode.yml -f prompt="审查代码"

# schedule (CRON)
on:
  schedule:
    - cron: '0 0 * * *'  # 每天午夜
```

### 10.5 工作流程文件

```yaml
# .github/workflows/opencode.yml
name: opencode

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  opencode:
    if: |
      contains(github.event.comment.body, ' /oc') ||
      startsWith(github.event.comment.body, '/oc')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: read
      issues: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Run opencode
        uses: anomalyco/opencode/github@latest
        with:
          model: anthropic/claude-sonnet-4-20250514
```

### 10.6 核心功能

- **权限检查**: 验证用户是否有写入权限
- **评论反应**: 添加 👀 表情表示正在处理
- **分支管理**: 自动创建分支、提交代码
- **PR 创建**: 自动创建 Pull Request
- **图像支持**: 支持下载 GitHub 附件中的图像
- **结果分享**: 生成 opencode.ai 分享链接

### 10.7 认证方式

| 方式             | 配置                   | 说明         |
| ---------------- | ---------------------- | ------------ |
| **OIDC**         | `id-token: write` 权限 | 默认，推荐   |
| **GitHub Token** | `GITHUB_TOKEN`         | 使用用户令牌 |
| **PAT**          | `github_pat_***`       | 个人访问令牌 |
| **AWS OIDC**     | Amazon Bedrock         | 云服务商认证 |

---

## 11. API 调用链

### 11.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  CLI/TUI    │  │   Web App   │  │    Desktop (Tauri)     │ │
│  │             │  │ (SolidJS)   │  │                         │ │
│  └──────┬──────┘  └──────┬──────┘  └────────────┬────────────┘ │
└─────────┼────────────────┼─────────────────────┼───────────────┘
          │                │                      │
          ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Local Server (Bun/Hono)                        │
│             http://localhost:4096                                 │
├─────────────────────────────────────────────────────────────────┤
│ Routes:                                                        │
│  /session/*    - 会话管理                                        │
│  /project/*   - 项目管理                                         │
│  /provider/*  - AI Provider                                     │
│  /config/*    - 配置                                            │
│  /file/*      - 文件操作                                        │
│  /mcp/*       - MCP 服务器                                      │
│  /pty/*       - 终端                                           │
│  /tui/*       - TUI 路由                                        │
├─────────────────────────────────────────────────────────────────┤
│  Core Services:                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
│  │  Session/Msg    │  │  LLM Stream     │  │  Tool Executor │ │
│  └─────────────────┘  └─────────────────┘  └────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  LSP Client ◄────── stdio ──────► LSP Server (本地进程) │  │
│  │  - pyright, typescript, clangd, gopls, rust-analyzer...  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 主要路由

| 路由          | 功能        |
| ------------- | ----------- |
| `/session/*`  | 会话管理    |
| `/project/*`  | 项目管理    |
| `/provider/*` | AI Provider |
| `/config/*`   | 配置        |
| `/file/*`     | 文件操作    |
| `/mcp/*`      | MCP 服务器  |
| `/pty/*`      | 终端 (PTY)  |
| `/tui/*`      | TUI 路由    |

---

## 12. MCP (Model Context Protocol) 支持

### 12.1 传输方式

- **stdio**: 本地进程
- **SSE**: HTTP 流
- **StreamableHTTP**: 流式 HTTP

### 12.2 功能

- 外部 MCP 服务器连接
- OAuth 认证
- 工具调用转发
- 动态工具列表

---

## 13. 配置文件

### 13.1 配置路径

| 平台        | 路径                  |
| ----------- | --------------------- |
| Linux/macOS | `~/.config/opencode/` |
| Windows     | `%APPDATA%\opencode\` |

### 13.2 配置结构

```json
{
  "model": "opencode/big-pickle",
  "provider": {
    "opencode": { "options": {} },
    "anthropic": { "env": ["ANTHROPIC_API_KEY"] },
    "google": { "npm": "@ai-sdk/google", "models": { ... } }
  },
  "agent": { ... },
  "permission": { ... },
  "plugin": [ ... ],
  "mcp": { ... }
}
```

---

## 14. 关键技术栈

| 层级     | 技术                         |
| -------- | ---------------------------- |
| 运行时   | Bun                          |
| Web 框架 | Hono                         |
| AI SDK   | Vercel AI SDK (`ai`)         |
| 数据库   | SQLite + Drizzle ORM         |
| UI 框架  | SolidJS                      |
| 桌面     | Tauri                        |
| 云函数   | Cloudflare Workers           |
| LSP      | vscode-jsonrpc               |
| 认证     | OAuth 2.0, API Key           |
| 协议     | MCP (Model Context Protocol) |

---

## 15. 版本与发布

### 15.1 构建命令

```bash
# CLI
cd packages/opencode && bun run build

# SDK
./packages/sdk/js/script/build.ts

# Desktop
bun run dev:desktop

# Web
bun run dev:web
```

### 15.2 测试

```bash
# 运行测试
bun test

# 单个测试文件
bun test path/to/test.test.ts

# 类型检查
cd packages/opencode && bun run typecheck
```
