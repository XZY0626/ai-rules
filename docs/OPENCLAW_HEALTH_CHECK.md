# OpenClaw 健康检查与运维手册

> **适用版本**：OpenClaw v2026.3.x  
> **最后更新**：2026-03-15  
> **维护者**：WorkBuddy（自动维护）

---

## 📌 概述

本文档记录 OpenClaw gateway 的标准健康检查流程、常见故障模式及修复方案。适用于：

- 日常巡检（建议每次大版本升级后执行）
- 排查故障时的快速参考
- AI 助手（WorkBuddy/龙虾）执行运维任务时的 SOP

---

## 🔍 健康检查清单

以下为完整巡检项目，按优先级排列。

### 1. 服务与端口

```bash
# Gateway 服务状态
systemctl --user status openclaw-gateway

# 端口监听确认
ss -tlnp | grep 18789

# Tailscale 状态
tailscale status
```

**期望结果**：
- gateway：`active (running)`
- 端口 18789：`LISTEN`
- Tailscale：`online`，可访问 HTTPS 地址

---

### 2. 版本一致性

```bash
# Gateway 实际使用的版本（读取 package.json）
cat ~/.local/lib/openclaw/package.json | grep '"version"'

# CLI 版本（应与 gateway 一致）
~/.local/bin/openclaw version

# 系统 PATH 中的 openclaw 指向哪里
which openclaw
```

**常见问题**：`/usr/bin/openclaw`（系统包管理器安装的旧版）优先于 `~/.local/bin/openclaw`（手动安装的新版）。

**修复方式**：在 `~/.bashrc` 追加 `export PATH="$HOME/.local/bin:$PATH"`。

---

### 3. Memory Search 状态

```bash
~/.local/bin/openclaw memory status
```

**期望结果**：
```
Provider: openai (requested: openai)
Model: text-embedding-v3
Vector: ready
FTS: ready
```

**常见问题及修复**：

#### 问题 A：`provider: none (requested: openai)`

原因：`auth-profiles.json` 不存在或格式错误。

**正确的 `auth-profiles.json` 格式**（`AuthProfileStore` 规范）：

```json
{
  "version": 1,
  "profiles": {
    "qwen-embedding": {
      "type": "api_key",
      "provider": "openai",
      "key": "<YOUR_API_KEY>"
    }
  }
}
```

文件路径：`~/.openclaw/agents/main/agent/auth-profiles.json`

> ⚠️ **注意**：`{"openai": {"apiKey": "..."}}` 这种格式是**错误的**，不会被识别。

#### 问题 B：`memorySearch` 块配置缺失或位置错误

> ⚠️ **关键坑**：`memorySearch` 配置必须写在 `agents.defaults.memorySearch` 下，**顶层的 `memorySearch` 已被废弃**（v2026.x 自动迁移）。

在 `~/.openclaw/openclaw.json` 的正确位置：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-v3",
        "remote": {
          "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
          "apiKey": "<YOUR_QWEN_API_KEY>"
        },
        "extraPaths": ["/home/<用户>/.openclaw/workspace"]
      }
    }
  }
}
```

> ⚠️ **不要再用 `auth-profiles.json` 存 embedding API Key**——那个文件是给模型对话用的，不是给 Memory Search 用的。

**`extraPaths` 的作用**：默认只索引 `memory/` 子目录。加上 `extraPaths` 才能把 workspace 根目录的 AGENTS.md、AI_RULES.md 等文件纳入语义搜索范围。

**`remote.baseUrl` 的作用**：不设置则默认请求 `api.openai.com`，导致 `fetch failed`。阿里云 Qwen embedding 必须显式设置。

> 阿里云 Qwen 的 embedding API 走 OpenAI 兼容接口，`provider` 填 `"openai"`（不是 `"dashscope"`）。

**支持的 provider 值**（来自 OpenClaw 源码枚举）：
`openai` | `local` | `gemini` | `voyage` | `mistral` | `ollama` | `auto`

---

### 4. systemd 服务配置

Gateway 服务文件路径：`~/.config/systemd/user/openclaw-gateway.service`

**推荐配置（包含性能优化）**：

```ini
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
ExecStartPre=/usr/bin/mkdir -p /var/tmp/openclaw-compile-cache
ExecStart=/usr/bin/node /home/<用户>/.local/lib/openclaw/dist/index.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Environment=OPENCLAW_NO_RESPAWN=1

[Install]
WantedBy=default.target
```

**性能变量说明**：

| 变量 | 作用 |
|------|------|
| `NODE_COMPILE_CACHE` | Node.js V8 字节码缓存，减少重复 CLI 调用的启动时间 |
| `OPENCLAW_NO_RESPAWN=1` | 禁止 gateway 内部自重启，避免 VM 低功耗环境的额外开销 |

修改后执行：

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

---

### 5. Cron 任务检查

```bash
~/.local/bin/openclaw cron list
```

**常见警告**：

```
Legacy cron job storage detected at ~/.openclaw/cron/jobs.json.
- 1 job needs payload kind normalization
Repair with: openclaw doctor --fix
```

此警告不影响实际运行，可在下次版本升级时随版本迁移自动解决。若要手动修复，执行：

```bash
~/.local/bin/openclaw doctor --fix
```

---

### 6. Doctor 全量诊断

```bash
~/.local/bin/openclaw doctor
```

常见输出条目说明：

| 诊断项 | 正常状态 | 异常处理 |
|--------|---------|---------|
| `config schema` | `ok` | 检查 openclaw.json 字段合法性 |
| `memory search` | `provider: openai, vector: ready` | 见上方 Memory Search 章节 |
| `cron jobs` | `ok` 或 legacy format 警告 | 执行 `doctor --fix` |
| `orphan transcripts` | `0 orphan files` | 执行 `doctor --fix` 或手动清理 |
| `gateway connectivity` | `reachable` | 检查服务是否正常启动 |

---

### 7. 孤立 Transcript 文件清理

Transcript 文件存放于 `~/.openclaw/agents/main/sessions/`。随着会话积累，`sessions.json` 中未索引的孤立文件会占用磁盘空间。

**检查**：

```bash
# 当前文件总数
ls ~/.openclaw/agents/main/sessions/ | wc -l

# sessions.json 中的活跃 session 数
cat ~/.openclaw/agents/main/sessions/sessions.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d))"
```

**清理**（仅删除 .deleted 和 .reset 后缀）：

```bash
find ~/.openclaw/agents/main/sessions/ -name "*.deleted" -o -name "*.reset" | xargs rm -f
```

> ⚠️ **不要批量删除所有 .jsonl 文件**，活跃会话的 transcript 也是 `.jsonl` 格式。

---

## 🔧 一键快速诊断脚本

将以下脚本保存为 `~/.local/bin/oc-health`（chmod +x）：

```bash
#!/bin/bash
echo "=== OpenClaw Health Check ==="
echo ""
echo "--- 1. Gateway Service ---"
systemctl --user status openclaw-gateway --no-pager | head -5

echo ""
echo "--- 2. Port 18789 ---"
ss -tlnp | grep 18789 || echo "NOT LISTENING"

echo ""
echo "--- 3. CLI Version ---"
~/.local/bin/openclaw version 2>/dev/null || echo "version cmd not available"

echo ""
echo "--- 4. Memory Search ---"
~/.local/bin/openclaw memory status 2>&1 | grep -E "Provider|Model|Vector|FTS"

echo ""
echo "--- 5. Cron Jobs ---"
~/.local/bin/openclaw cron list 2>&1 | head -10

echo ""
echo "--- 6. Doctor Summary ---"
~/.local/bin/openclaw doctor 2>&1 | grep -E "ok|warn|error|WARN|ERROR" | head -15

echo ""
echo "=== Done ==="
```

---

## 📋 已知问题与历史修复

| 日期 | 问题 | 状态 | 修复方案 |
|------|------|------|---------|
| 2026-03-15 | Memory Search `remote.baseUrl` 未设置，请求打 api.openai.com → `fetch failed` | ✅ 已修复 | 在 `agents.defaults.memorySearch.remote` 写入 baseUrl + apiKey |
| 2026-03-15 | Memory Search 只索引 `memory/` 子目录，workspace 根目录文件未被索引 | ✅ 已修复 | 设置 `extraPaths: ["/home/xzy0626/.openclaw/workspace"]` |
| 2026-03-15 | systemd 缺少 NODE_COMPILE_CACHE 性能变量 | ✅ 已修复 | 写入 [Service] Environment 字段 |
| 2026-03-15 | CLI PATH 指向旧版 /usr/bin/openclaw | ✅ 已修复 | ~/.bashrc 追加 ~/.local/bin 优先路径 |
| 2026-03-14 | gateway 从 nohup 迁移到 systemd 用户级服务 | ✅ 已修复 | ~/.config/systemd/user/openclaw-gateway.service |
| 2026-03-13 | tailscale-serve 与 openclaw-gateway 启动顺序竞争 | ✅ 已修复 | tailscale-serve.service 加 ExecStartPre 等待网络就绪 |
| 2026-03-13 | gateway.bind=loopback 导致外部无法访问 | ✅ 已修复 | 通过 Tailscale serve 代理，外部走 Tailscale HTTPS |
| 2026-03-15 | tailscale-serve systemctl 显示 inactive 误以为服务挂了 | ✅ 已确认正常 | 该服务为 oneshot 类型，ExecStart 执行后正常退出，`tailscale serve --bg` 后台持续运行，HTTPS 代理正常工作 |

---

## 🔒 安全注意事项

1. **API Key 零接触原则**：`auth-profiles.json` 中的 key 字段由用户手动填写，AI 不读取、不输出、不在日志中显示其值
2. **gateway.bind=loopback**：对外不暴露 18789 端口，只通过 Tailscale HTTPS 访问
3. **SSH 私钥**：`E:\Application\WorkBuddy\id_vm` 权限严格限制，仅允许当前用户读写
4. **Tailscale 域名**：在公开文档中脱敏为 `[TAILSCALE_HOSTNAME].[TAILNET_DOMAIN]`
5. **认证标志文件检查（重要）**：每次巡检时必须确认 `~/.openclaw/` 下**不存在**以下文件：
   - `.pre-disable-auth` — 存在时 gateway 启动会跳过认证，任何进程均可无凭证访问所有 API
   - 任何以 `.disable-` 或 `.skip-` 开头的标志文件
   ```bash
   ls ~/.openclaw/.pre-disable-auth 2>/dev/null && echo "⚠️ 警告：认证旁路文件存在！" || echo "✅ 认证旁路文件不存在"
   ```
   若文件存在：立即执行 `rm -f ~/.openclaw/.pre-disable-auth` 并重启 gateway，同时向用户告警

---

## 🔗 相关文档

- [AI_RULES.md L1.4 - OpenClaw 运维安全规范](../AI_RULES.md)
- [操作日志 - 20260315 全面健康检查](https://github.com/XZY0626/ai-session-logs/blob/main/logs/workbuddy/2026-03/20260315-openclaw-full-health-check.md)
- [操作日志 - 20260314 OpenClaw 升级 v2026.3.13](https://github.com/XZY0626/ai-session-logs/blob/main/logs/openclaw/2026-03/20260314-openclaw-upgrade-3.11-to-3.13.md)
