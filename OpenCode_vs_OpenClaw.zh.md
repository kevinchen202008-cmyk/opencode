### OpenCode vs OpenClaw 架构对比与组合方案（简体中文）

> 本文基于本仓库中的 `ARCHITECTURE.zh.md`（OpenCode）与 `ARCHITECTURE.openclaw.zh.md`（OpenClaw）两份架构文档进行对比，并提出一个二者结合使用的创意解决方案。

---

### 一、定位与核心场景对比

| 维度 | OpenCode | OpenClaw |
| ---- | -------- | -------- |
| 核心定位 | 开源 AI Coding Agent，面向开发者和代码工作流 | 个人 AI 助手，面向消息渠道聚合和设备控制 |
| 主要用户 | 开发者 / 团队工程师 | 个人用户 / 小团队助手运营者 |
| 典型场景 | 读写代码、重构项目、运行测试、调用 MCP 工具 | 跨渠道收发消息、自动回复、定时任务、浏览器自动化、控制桌面与手机 |
| 交互界面 | TUI / Desktop / Web / VSCode 扩展 / SDK | WhatsApp / Telegram / Slack / Discord / WebChat / macOS App / iOS / Android / CLI |

---

### 二、宏观架构与“中枢”角色

| 维度 | OpenCode | OpenClaw |
| ---- | -------- | -------- |
| 核心中枢 | OpenCode Server（HTTP + SSE） | Gateway WebSocket 网络（ws://…） |
| 分层视角 | Infra → Core Service（Server/Provider/MCP/Tools）→ Domain（Agents/Permissions/Plugins）→ Clients | Channels → Gateway（sessions / routing / cron / webhooks）→ Agent/Tools/Nodes → Clients & Nodes |
| 配置主入口 | `opencode.json` + `.opencode/` | Gateway 配置（Nix/Docker/文件）+ Channels/Nodes/Skills 配置 |

---

### 三、Agent / 模型 / 工具 层对比

| 维度 | OpenCode | OpenClaw |
| ---- | -------- | -------- |
| Agent 抽象 | 每个 Agent = 模型 + 指令 + 工具列表 + 权限（allow/deny/ask） | Pi Agent Runtime：围绕会话和渠道进行多 Agent 路由 |
| 模型管理 | Provider 抽象层：多家 LLM + 本地模型，面向编码场景优化 | Models + Auth Profile + Failover：保证在线助手的稳定性与安全性 |
| 工具类型 | 文件（read/write/edit/apply_patch）/ grep / bash / MCP / LSP / webfetch / websearch 等 | Tools（browser/canvas/cron/webhooks/skills）+ Nodes（camera/screen/location/system.run）+ Channels/Groups 自动化 |
| 典型输出 | 代码变更、补丁、重构结果、测试与日志 | 消息回复、定时通知、浏览器/Canvas 操作、设备指令 |

---

### 四、安全与权限模型对比

| 维度 | OpenCode | OpenClaw |
| ---- | -------- | -------- |
| 安全焦点 | 防止错误/危险工具调用（特别是文件写入与 Shell） | 防止不可信 DM/群组输入导致高危操作 |
| 权限机制 | `permission` 字段控制每个工具或前缀（allow/deny/ask） | DM pairing + allowlist、Channel/Group 路由规则、模型 failover、安全文档与 doctor 工具 |
| MCP 策略 | 原生 MCP 支持（本地+远程），作为工具来源之一 | 通过 `mcporter` 桥接 MCP，保持核心 Gateway 精简 \(`VISION.md`\) |

---

### 五、扩展与生态对比

| 维度 | OpenCode | OpenClaw |
| ---- | -------- | -------- |
| 插件/扩展 | 插件系统（`@opencode-ai/plugin`），自定义 Tools / Agents / Skills / Commands | 插件 API + Skills 平台 + ClawHub 社区技能 |
| MCP 集成 | 核心内置，适合挂企业内部服务（Issue、CI、监控、DB 等） | 通过 mcporter 统一收编 MCP，优先保持核心安全与稳定 |
| 生态优势 | 更适合工程团队把内部开发/运维系统接入 AI 代码助手 | 更适合围绕「个人+组织消息渠道」构建自动化助手生态 |

---

### 六、创意组合方案：OpenClaw 作为外部助手入口，OpenCode 作为开发工作后端

#### 6.1 总体思路

- **OpenClaw**：统一处理所有外部输入（消息渠道、移动节点、桌面节点等），作为「外部世界」与用户的主交互层。
- **OpenCode**：专注在具体项目或代码仓库中执行「开发工作流」，作为「开发工作后端」被 OpenClaw 调用。
- 用户只需要在手机/聊天工具里对 OpenClaw 说话，就能间接驱动 OpenCode 在某个代码仓库里读写代码、跑测试，并把结果回传到聊天渠道。

可以理解为：

- OpenClaw = 外部多渠道 **统一入口 + 调度中心**。
- OpenCode = 针对代码仓库的 **专业执行引擎**。

#### 6.2 组合架构图（概念级）

```mermaid
flowchart TB
  subgraph Channels[外部渠道与设备]
    WA[WhatsApp / Telegram / Slack / ...]
    Phone["iOS / Android 节点\n(Voice / Camera / Screen)"]
    MacApp[macOS App]
  end

  subgraph Gateway[OpenClaw Gateway\n(Control Plane)]
    GW[Gateway WS\nsessions / routing / cron / webhooks]
    Pi[Pi Agent Runtime]
    Skills[Skills / Plugins\n(OpenCode Integration Skill)]
  end

  subgraph DevHost[开发主机 / CI]
    OCServer["OpenCode Server\n(opencode serve)"]
    Repo["代码仓库\n(Workspace / Git Repo)"]
  end

  subgraph UserIDE[可选：本地 IDE]
    IDE["VSCode / Cursor\n+ OpenCode 扩展"]
  end

  WA --> GW
  Phone --> GW
  MacApp --> GW

  GW --> Pi
  Pi --> Skills
  Skills -->|HTTP / SDK / CLI| OCServer
  OCServer --> Repo

  IDE -->|HTTP / SSE| OCServer
```

解释：

- 所有聊天输入（比如「帮我给 repo X 修一下登录 bug」）进入 OpenClaw Gateway。
- Gateway 根据配置将这类「开发任务」路由给一个专门的 **OpenCode Integration Skill**。
- 该 Skill 通过 HTTP/SDK/CLI 调用 OpenCode Server，在指定仓库上执行一系列开发操作。
- OpenCode 执行完后返回摘要/日志/补丁信息给 Skill，再由 OpenClaw 回复到用户的聊天渠道。
- 同时，开发者可以在本地 IDE（VSCode/Cursor）中通过 OpenCode 看到相同的会话与变更，实现「手机提需求、IDE 看/改细节」的协同。

#### 6.3 关键集成点设计

1. **OpenClaw 侧：OpenCode Integration Skill**
   - 以 Skill / Plugin 形式存在：
     - 暴露高层指令，例如：`fix_bug(repo, description)`、`add_feature(repo, spec)`、`refactor_module(repo, target)`。
   - 内部实现：
     - 解析用户自然语言（目标仓库/分支/模块/需求）。
     - 调用 OpenCode Server 的 HTTP API 或通过 CLI + RPC 触发：
       - 创建/选择项目 Session。
       - 把需求转写成 OpenCode Agent 的系统指令 + 用户消息。
       - 订阅 OpenCode 的进度事件（可选）。

2. **OpenCode 侧：面向 OpenClaw 的 Agent 模板**
   - 定义一个专用 Agent，如 `claw_bridge`：
     - 指令专门面向「从 OpenClaw 收到的需求」，输出风格以「简洁摘要 + 可读补丁说明」为主。
     - 工具权限适当收紧（例如禁止任意 Shell，只允许必要的 git / test 命令），以适应「远程触发」场景。

3. **通信方式建议**
   - 简单版本：OpenClaw Skill 以 HTTP/REST 调用 OpenCode Server：
     - `POST /project` / `POST /session` / `POST /session/:id/message` 等。
   - 稍复杂版本：通过一个轻量的中间服务，把 OpenCode 的 SSE 事件流转成 WebSocket / Webhook 推给 OpenClaw，支持进度更新和长任务跟踪。

4. **权限与安全控制**
   - OpenClaw 侧：
     - Skill 只对特定「受信用户」开放（例如只接受主人账号的开发请求）。
     - 对每次跨系统调用增加显式确认（如「是否允许我在 repo X 上进行代码修改？」）。
   - OpenCode 侧：
     - 为 `claw_bridge` Agent 设置严格的 Tool 权限和访问路径（限定在某些仓库根目录内）。
     - 可以通过 OpenCode 的权限系统要求对危险操作（如 `bash`）进行人工确认。

#### 6.4 示例时序（文字化）

1. 用户在 Telegram 给 OpenClaw 发送消息：「帮我在 `project-a` 里把登录失败重试次数从 3 改成 5，并更新相关文档」。
2. Telegram 适配器将消息送入 Gateway，路由到对应的个人 Agent。
3. 该 Agent 识别出「开发任务」，调用 OpenCode Integration Skill。
4. Skill：
   - 基于配置找到 `project-a` 所在的工作目录/仓库。
   - 构造一条 OpenCode 会话消息并调用 OpenCode Server（选择 `claw_bridge` Agent）。
5. OpenCode：
   - 使用文件/grep 工具找到登录逻辑及相关配置。
   - 调整重试次数，更新文档，运行测试。
   - 返回变更摘要与关键 DIFF（或提交 ID）。
6. Skill 收到结果后，将「自然语言摘要 + 可选的代码片段链接」通过 Gateway 回复到 Telegram。
7. 用户如有需要，可以在本地 IDE 中打开同一个仓库，查看 OpenCode 已完成的更改并做最后 Review。

---

### 七、总结

- **OpenCode** 更像是「专业开发后端」：擅长在本地代码仓库中进行复杂、受控的自动化开发任务。
- **OpenClaw** 更像是「外部世界入口与生活/工作助手」：聚合多渠道消息与设备，适合面向非开发环境的交互。
- 通过一个 **OpenClaw Skill ↔ OpenCode Agent** 的桥接方案，可以让用户在日常聊天工具中**自然地发起开发任务**，再由 OpenCode 在代码环境中专业执行，从而实现「手机发需求、IDE 看结果」的端到端闭环。

