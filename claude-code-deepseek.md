---
name: claude-code-deepseek
description: "在 Ubuntu 无头服务器上安装 Claude Code CLI 并配置 DeepSeek V4 Pro/Flash 后端，支持 Print 模式与交互式模式"
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [claude-code, deepseek, coding-agent, cli, ubuntu]
    related_skills: [hermes-agent, ubuntu-proxy-singbox]
---

# Claude Code + DeepSeek 配置指南

在 Ubuntu 无头服务器上安装 Claude Code CLI，通过 DeepSeek 原生 Anthropic 兼容端点 `https://api.deepseek.com/anthropic` 接入 DeepSeek V4 Pro / Flash 模型，无需 Anthropic 官方订阅。

## 适用场景

- 需要在无头服务器（无桌面环境）上使用 Claude Code 编码代理
- 希望使用 DeepSeek 模型而非 Anthropic 官方付费订阅
- 既要从 Hermes Agent 自动调用（Print 模式），也要自行 SSH 交互式使用

## 前置条件

- **Node.js ≥ 18**（安装 Claude Code CLI 所需）
- **DeepSeek API Key**（从 DeepSeek 平台获取）
- 网络可访问 `api.deepseek.com`

## 安装步骤

### 1. 安装 Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

验证安装：

```bash
claude --version
# 预期输出: 2.x.xxx (Claude Code)
```

### 2. 配置 DeepSeek 环境变量

DeepSeek 提供了原生 Anthropic 兼容端点 `/anthropic`，无需任何格式转换代理。

将以下内容加入 `~/.bashrc`：

```bash
export DEEPSEEK_API_KEY=sk-your-deepseek-api-key-here
alias claude-deepseek='ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic ANTHROPIC_API_KEY=$DEEPSEEK_API_KEY claude'
alias claude-anthropic='claude'
```

生效：

```bash
source ~/.bashrc
```

### 3. 可用模型

| 模型 | 身份 | 适用场景 |
|------|------|---------|
| `deepseek-v4-pro` | 主力 | 复杂编码、重构、多步任务 |
| `deepseek-v4-flash` | 快速 | 简单任务、代码审查、快速原型 |

通过环境变量切换模型：

```bash
ANTHROPIC_MODEL=deepseek-v4-flash claude-deepseek -p "快速检查代码"
```

## 使用方式

### Print 模式（非交互式，Hermes 调用）

一次性任务，执行完毕自动退出。适合 Hermes Agent 通过 `delegate_task` 调用。

```bash
claude-deepseek -p "修复 src/api.py 中的认证 bug" --max-turns 10
```

常用参数：

| 参数 | 作用 |
|------|------|
| `-p "任务描述"` | Print 模式，非交互执行 |
| `--max-turns N` | 限制最大推理轮数（防止死循环） |
| `--output-format json` | 输出结构化 JSON（含耗时、花费、修改了啥） |
| `--model <name>` | 指定模型 |
| `--allowedTools "Read,Edit,Bash"` | 限制可用工具 |
| `--max-budget-usd N` | 限花费上限 |
| `-c, --continue` | 继续最近一次会话 |

### 交互式模式（你 SSH 直接使用）

完整的 REPL，支持多轮对话、斜杠命令。

```bash
# 启动交互式
claude-deepseek

# 或带初始提示启动
claude-deepseek "重构这个模块，考虑性能优化"
```

交互式支持的命令：

| 命令 | 作用 |
|------|------|
| `/review` | 代码审查 |
| `/plan` | 进入计划模式 |
| `/compact` | 压缩上下文节省 token |
| `/model` | 切换模型 |
| `/effort` | 调整推理深度 |
| `/exit` 或 `Ctrl+D` | 退出 |

### Hermes Agent 自动调用

在 Hermes 中通过 `delegate_task` 自动调用 Claude Code + DeepSeek：

```
delegate_task(
  goal="重构 src/api.py，添加错误处理",
  context="项目在 /home/project，用户偏好 type hints + Google docstring"
)
```

内部实际执行的命令为：

```bash
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic \
ANTHROPIC_API_KEY=$DEEPSEEK_API_KEY \
claude -p "重构 src/api.py" --max-turns 10 --allowedTools "Read,Edit,Bash"
```

## 其他支持的 Anthropic 兼容端点

以下提供商同样提供 Anthropic 兼容 API，可直接替换 `ANTHROPIC_BASE_URL`：

| 提供商 | 端点 | 模型示例 |
|--------|------|---------|
| **DeepSeek** | `https://api.deepseek.com/anthropic` | `deepseek-v4-pro` / `deepseek-v4-flash` |
| **MiniMax** | `https://api.minimax.io/anthropic` | `MiniMax-M2.7` |
| **Z.ai** | `https://api.z.ai/api/anthropic` | `glm-4.7` |
| **OpenRouter** | `https://openrouter.ai/api` | `deepseek/deepseek-v3.2`（需 Anthropic Skin） |
| **Synthetic** | `https://api.synthetic.new/anthropic` | `GLM-4.7` |

## 常见问题

### Q: 需要 Anthropic 账号吗？
不需要。通过 `ANTHROPIC_BASE_URL` 指向 DeepSeek 端点，完全绕过 Anthropic。

### Q: Print 模式和交互式模式有什么区别？
Print 模式一次执行完退出，适合自动化；交互式模式保持对话，适合你用 SSH 连上来慢慢聊。

### Q: `claude auth login` 需要执行吗？
不需要。走 DeepSeek 端点时通过 `ANTHROPIC_API_KEY` 环境变量认证，跳过 OAuth 登录流程。

### Q: 能同时保留官方 Anthropic 使用吗？
可以。`claude-anthropic` 走官方通道，`claude-deepseek` 走 DeepSeek，互不干扰。

## 验证清单

- [ ] `claude --version` 输出正常
- [ ] `claude-deepseek -p "你好"` 成功返回中文回答
- [ ] `claude-deepseek` 交互式模式正常启动
- [ ] `ANTHROPIC_MODEL=deepseek-v4-flash claude-deepseek -p "test"` 可切换模型
- [ ] `source ~/.bashrc` 后 alias 生效
