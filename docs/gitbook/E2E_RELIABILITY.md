# OpenCode E2E 可靠性分析报告

用户场景：Windows/Linux 执行机 + GitHub CI/CD + GLM Provider + CLI/VSCode

---

## 📐 用户部署架构

```
GitHub → Windows/Linux → OpenCode → GLM ↺ VSCode
  (CI/CD)  (执行机)    (Agent)    (LLM)   (IDE)
```

---

## 🔍 各环节可靠性分析

### 1. Windows/Linux 执行机

| 问题类型 | 严重程度 | 说明                          |
| -------- | -------- | ----------------------------- |
| 启动慢   | 🟡 中    | #14965, #7979 报告启动加载慢  |
| 内存问题 | 🔴 高    | 内存泄漏 70GB+，OOM kill      |
| Keybinds | 🟡 中    | #4997 Windows 键盘绑定问题    |
| TUI 冻结 | 🟡 中    | opentui 在 Windows 上随机冻结 |

### 2. GitHub CI/CD

| 功能       | 状态    |
| ---------- | ------- |
| Issue 触发 | ✅ 稳定 |
| PR 审查    | ✅ 稳定 |
| 定时任务   | ✅ 稳定 |
| OIDC 认证  | ✅ 稳定 |

### 3. GLM Provider

| Issue  | 问题                   | 严重程度 |
| ------ | ---------------------- | -------- |
| #15782 | GLM-5 reasoning 停止   | 🔴 高    |
| #10731 | GLM-4.7 解析问题       | 🟡 中    |
| #13900 | GLM-5 MCP JSON 错误    | 🔴 高    |
| #13310 | GLM-4.7 无法读取图片   | 🟡 中    |
| #11943 | GLM-4.7 卡在 plan 模式 | 🟡 中    |

### 4. CLI 模式

| 功能      | 状态    |
| --------- | ------- |
| 核心对话  | ✅ 稳定 |
| Tool 执行 | ✅ 稳定 |
| 文件操作  | ✅ 稳定 |
| LSP 支持  | 🟡 中等 |

### 5. VSCode 插件

| Issue  | 问题                  | 严重程度 |
| ------ | --------------------- | -------- |
| #15426 | 输出格式混乱          | 🟡 中    |
| #15643 | Activate.ps1 导致退出 | 🟡 中    |
| #11115 | 滚动严重延迟          | 🟡 中    |
| #14810 | shift+enter 键位问题  | 🟢 低    |
| #15461 | 文件路径格式错误      | 🟢 低    |

---

## ⚠️ E2E 可靠性风险

### 🔴 高风险点

- **GLM 模型稳定性** - 多起 tool call JSON 格式错误、reasoning 停止、plan 模式卡死
- **Windows 内存问题** - 长时间运行可能导致 OOM，尤其在 CI 环境中
- **执行机资源泄漏** - 孤儿进程未清理，可能耗尽 CI 执行机资源

### 🟠 中风险点

- **GLM 多模态** - 图片读取功能不完整
- **VSCode 插件** - Windows 上有兼容性问题
- **启动性能** - CI 环境下每次启动都需要等待

### ✅ 稳定点

- **GitHub CI/CD** - 核心触发机制成熟稳定
- **CLI 核心功能** - 对话、工具执行基本稳定
- **网络请求** - 与 GLM API 通信相对稳定

---

## 📊 E2E 可靠性评分

| 环节               | 评分       |
| ------------------ | ---------- |
| 执行机 (Win/Linux) | ⭐⭐⭐     |
| GitHub CI/CD       | ⭐⭐⭐⭐⭐ |
| GLM Provider       | ⭐⭐       |
| CLI 模式           | ⭐⭐⭐⭐   |
| VSCode 插件        | ⭐⭐⭐     |

---

## 💡 可靠性提升建议

### 针对 GLM Provider

1. 考虑添加 fallback provider (如 OpenAI/Anthropic)
2. 实现重试机制处理 JSON 解析失败
3. 监控 tool call 错误率
4. 避免在关键 CI 流程中使用 GLM-5 的 MCP 功能

### 针对 Windows/Linux 执行机

1. 为 CI 环境设置内存限制 (如 4GB)
2. 实现健康检查和自动重启
3. 使用 Docker 容器隔离执行环境
4. 监控进程数量，防止孤儿进程累积

### 针对 VSCode 插件

1. 生产环境优先使用 CLI 模式
2. Windows 上注意虚拟环境激活脚本问题
3. 关注插件版本更新

---

## 📝 总结

基于用户场景的 E2E 可靠性评估：

- ✅ **GitHub CI/CD** - 最稳定环节
- ✅ **CLI 模式** - 基本可用，推荐生产使用
- ⚠️ **GLM Provider** - 需要额外监控和 fallback
- ⚠️ **Windows 执行机** - 需关注内存和进程管理
- ⚠️ **VSCode 插件** - 建议用于开发，生产用 CLI

> **推荐配置：** Linux 执行机 + GitHub Actions + CLI + (GLM + OpenAI fallback)

---

📅 **分析日期**: 2026-03-03  
📊 **数据来源**: GitHub anomalyco/opencode Issues & PRs
