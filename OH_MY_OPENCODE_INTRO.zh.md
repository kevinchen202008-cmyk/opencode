### Oh My OpenCode 组织内使用说明（草案）

> 本文说明：oh-my-opencode 与 OpenCode 的关系、架构差异，以及在组织内**建议如何使用与管控**。配合 `OPENCODE_ORG_GUIDE.zh.md` 一起阅读。

---

### 一、它是什么？与 OpenCode 的关系

- **OpenCode**：开源 AI Coding Agent 平台，是「内核 / 引擎」：
  - 提供 Server、Agent、Tool、Provider、MCP、Plugin、Skills 等通用能力。
  - 偏「基础设施」，灵活但需要自己设计工作流与配置。
- **oh-my-opencode**：运行在 OpenCode 之上的**插件 + 配置 + Agent 编排层**：
  - 通过 OpenCode 的 **Plugin + Agent + Skill + MCP + Hook** 机制实现。
  - 自带一套「强意见」的默认配置和多 Agent 编排策略（Sisyphus、Oracle、Librarian、Explore、前端工程师等）。
  - 目标是让 OpenCode「开箱即用」，在多模型、多 Agent、复杂代码库场景下有较强生产力。

一句话：**OpenCode 是发动机，oh-my-opencode 是装在发动机上的「高配驾驶舱 + 自动挡 + 巡航系统」**。

---

### 二、架构与能力上的主要差异（简要对比）

| 维度 | OpenCode（本体） | oh-my-opencode（插件层） |
| ---- | ---------------- | ------------------------ |
| 角色 | 平台 / 内核 | 插件 / Agent Harness |
| 核心能力 | Server / Agent / Tool / MCP / Plugin / Skills | 多 Agent 协同、背景任务、LSP/AST 工具、预置 MCP 与 Skills、补充 Hook |
| 默认 Agents | build / plan / general / explore | Sisyphus（主）、Prometheus（Planner）、Oracle（架构/Debug）、Librarian（文档/源码）、Explore（快速搜索）、前端工程师等 |
| MCP & Skills | 提供机制，需自行配置 | 预集成 Exa、Context7、grep.app 等 MCP，内置 `git-master`、`playwright` 等 Skills |
| Hook 使用 | 官方插件与用户插件根据需要挂载 | 大量 Hook 预置（如 PreToolUse/PostToolUse/UserPromptSubmit/Stop），用于注入规则、强制 Todo 执行、注释检查等 |

从组织视角看：**OpenCode 提供能力边界，oh-my-opencode 提供一套实战工作方法和默认策略**。

---

### 三、在组织内的推荐使用方式

#### 3.1 环境策略

- **dev 环境**：
  - 推荐安装 oh-my-opencode，体验其多 Agent 协同与高效工作流。
  - 允许开发者在个人/团队仓库中试验、调整和裁剪其配置。
- **staging 环境**：
  - 仅启用经过审查的功能子集（受控的 Agents、MCP、Skills 与 Hooks）。
  - 用于验证 oh-my-opencode 对典型项目的实际收益与安全性。
- **prod 环境**：
  - 谨慎选择是否启用 oh-my-opencode：
    - 若启用，**只启用通过安全/合规评审的配置与子集**。
    - 或选择「吸收理念、自建轻量插件」的方式，不直接部署原版。

#### 3.2 能力白名单思路

在组织内，可以把 oh-my-opencode 的能力拆成几类，分别决策：

- **强推荐启用（在 dev/staging）**：
  - 多 Agent 规划与分工（Sisyphus + Planner/Oracle/Librarian/Explore）。
  - LSP/AST 工具、重构/诊断增强。
  - Comment Checker / Todo Enforcer 等「代码质量与任务完成度」增强。
- **条件启用（需评估合规性）**：
  - 内置 MCP（Exa、Context7、grep.app 等），涉及外网访问与代码/文档上传。
  - `git-master` 等自动化 Git 操作型 Skill，在生产仓库上需额外保护。
- **默认关闭（如不满足组织合规要求）**：
  - 任何可能绕过官方 ToS 的第三方插件或 OAuth 集成。
  - 对生产环境有直接高权限操作能力的 Hook 或 MCP（如云控制面、生产 DB 写入等）。

---

### 四、配置与管控建议

#### 4.1 配置入口

- 全局配置：`~/.config/opencode/opencode.json{c}` 中：
  - 在 `plugin` 列表中加入 `"oh-my-opencode"`。
  - 在专门的 `oh-my-opencode` 配置文件中（如 `~/.config/opencode/oh-my-opencode.json`）细化：
    - 默认启用的 Agents 与其模型/温度/权限。
    - 启用/禁用的 MCP 列表。
    - 开启/关闭的 Hooks。
    - 并发与任务队列限制。
- 项目级配置：`.opencode/oh-my-opencode.json`：
  - 调整特定项目需要的 Agent/Skill 集合。
  - 为大仓库调优并发与上下文注入策略。

#### 4.2 与组织级规范对齐

- 结合 `OPENCODE_ORG_GUIDE.zh.md` 中的组织策略，针对 oh-my-opencode 额外约束：
  - **权限**：利用 OpenCode 的 `permission` 与 Agent Frontmatter 精细控制：
    - 限制 `bash`、写文件、MCP 调用等高危工具的使用范围与行为。
  - **插件与 Hooks**：
    - 只启用通过代码审查与安全评估的 Hook 集合。
    - 根据需要在配置中 `disabled_hooks` 某些不符合内部规范的特性。
  - **MCP 与外部访问**：
    - 在内网部署环境中，评估是否允许访问公共 MCP（如 Exa、grep.app 等）。
    - 对涉及源代码或敏感文档的外发行为，按公司安全策略处理（如脱敏、白名单域名等）。

---

### 五、典型落地路径（建议）

1. **试点阶段（dev）**：
   - 在开发环境为部分自愿团队安装 oh-my-opencode。
   - 收集反馈：效率提升点、坑点、与现有流程的冲突。
2. **评估与收敛（staging）**：
   - 基于反馈与组织安全要求，梳理出：
     - 必须保留的特性（如多 Agent 协同、LSP/AST 工具）。
     - 必须关闭或替换的特性（如某些 MCP、某些 Hook）。
   - 固化成一份「公司版 oh-my-opencode 配置模板」。
3. **推广或内化（prod）**：
   - 方案 A：在生产环境部署「裁剪版 oh-my-opencode」，仅启用已经验证安全的部分。
   - 方案 B：将 oh-my-opencode 的配置与想法吸收为自建插件/配置集，在内部维护独立的 Harness。
4. **持续演进**：
   - 定期跟进上游更新（特别是 bugfix 与安全相关更改）。
   - 通过 PR/issue 与社区保持互动，同时在内部仓库中记录「我们对默认配置的偏离点」。

---

### 六、小结

- **OpenCode** 提供了强大的底层能力，但需要组织自己设计「怎么用」。
- **oh-my-opencode** 在此之上提供了高度工程化、多 Agent 协同的「使用方法」，可以极大提升个人与小团队生产力。
- 在组织级场景下，建议：
  - 把 oh-my-opencode 视为一个**可学习、可借鉴、可选择性采用的参考实现**。
  - 通过 dev/staging 环境试验和裁剪，最终形成「组织自己的 Harness 与配置规范」。

