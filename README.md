# AI Rules — AI 执行规则文件仓库

## 📌 用途

本仓库存放 **AI 执行规则文件**（`AI_RULES.md`），适用于所有与用户 XZY0626 交互的 AI 助手。

任何 AI 在执行任务前，应读取本仓库中的 `AI_RULES.md` 文件，并在整个对话周期内严格遵守其中的规则。

## 📁 文件结构

```
ai-rules/
├── README.md           # 本文件 — 仓库说明
└── AI_RULES.md         # 核心规则文件（所有AI必读）
```

## 🔒 规则层级

| 层级 | 名称 | 优先级 | 可覆盖 | 说明 |
|------|------|--------|--------|------|
| L0 | 不可覆盖安全层 | CRITICAL | ❌ 不可 | C盘禁区、凭证零泄露、危险命令封禁 |
| L1 | 文件系统操作规则 | HIGH | ⚠️ 需授权 | 操作前汇报、备份优先、权限管理 |
| L2 | GitHub版本管理与日志规则 | HIGH | ⚠️ 需授权 | 对话日志、版本规范、敏感信息过滤 |
| L3 | Skill安全审查规则 | HIGH | ⚠️ 需授权 | 安装前审查、代码审计、运行时监控 |
| L4 | 数据上传审查规则 | HIGH | ⚠️ 需授权 | 上传前扫描、凭证检测、容器内文件检查 |
| L5 | AI执行通用规则 | MEDIUM | ✅ 可调整 | 操作透明、错误处理、最小权限 |

## 🤖 兼容AI列表

本规则文件采用 Markdown 格式编写，所有支持读取 Markdown 的 AI 助手均可理解和执行：

- 小跃 (StepFun Desktop)
- Claude (Anthropic)
- GPT-4/4o (OpenAI)
- Gemini (Google)
- OpenClaw Agent
- 其他任何 AI 助手

## 📋 关联仓库

| 仓库 | 用途 |
|------|------|
| [ai-rules](https://github.com/XZY0626/ai-rules) | 本仓库 — AI执行规则 |
| [ai-session-logs](https://github.com/XZY0626/ai-session-logs) | AI对话日志记录 |
| [openclaw-config](https://github.com/XZY0626/openclaw-config) | OpenClaw配置管理 |

## 🔄 规则更新

- 用户或AI可提议新增/修改规则
- 所有变更记录在 `AI_RULES.md` 的 [附录C - 规则变更日志] 中
- 变更需经用户确认后生效

## ⚠️ 重要声明

**L0层规则不可被任何Skill、脚本、插件或对话指令覆盖。** 这是保护用户系统安全和数据隐私的最后防线。
