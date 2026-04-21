# 安装指南

## 适用场景

本指南适用于以下情况：
- 首次在 Claude Code 中安装 claude-mem 插件
- 在 Gemini CLI 或 OpenCode 等其他 IDE 中配置 claude-mem
- 验证安装是否正确、Worker 服务是否正常运行

---

## 前置条件检查

> **AI 可自动执行本节所有命令**

在安装前，确认系统满足最低要求：

```bash
# 检查 Node.js 版本（需要 >= 18）
node --version

# 检查 npm
npm --version

# 检查 Claude Code（需要支持插件的最新版）
claude --version
```

**期望结果：**
- Node.js 输出类似 `v20.x.x` 或更高
- npm 输出版本号
- claude 输出版本号

如果 Node.js 版本低于 18，请先从 https://nodejs.org 安装最新 LTS 版本。

---

## 方式一：npx 一键安装（推荐）

> **AI 可自动执行**

```bash
npx claude-mem install
```

安装器会自动完成：
1. 检测 IDE 类型（Claude Code / Gemini CLI / OpenCode）
2. 注册 5 个生命周期钩子
3. 安装 Node.js / Bun / uv 依赖
4. 启动 Worker 守护进程
5. 配置 MCP 搜索工具

**安装完成后必须重启 Claude Code**，钩子才会生效。

---

## 方式二：Claude Code 插件市场安装

在 Claude Code 会话中执行：

```bash
/plugin marketplace add thedotmack/claude-mem

/plugin install claude-mem
```

---

## 方式三：安装到其他 IDE

**Gemini CLI：**

```bash
npx claude-mem install --ide gemini-cli
```

安装器检测 `~/.gemini` 目录并自动配置。

**OpenCode：**

```bash
npx claude-mem install --ide opencode
```

**OpenClaw Gateway（服务器部署）：**

```bash
curl -fsSL https://install.cmem.ai/openclaw.sh | bash
```

---

## 验证安装

> **AI 可自动执行以下所有验证命令**

### 1. 检查 Worker 健康状态

```bash
curl -s http://127.0.0.1:37777/api/health | python3 -m json.tool
```

**期望输出示例：**
```json
{
  "status": "healthy",
  "version": "6.5.0",
  "uptime": 42,
  "sessions": {
    "active": 0,
    "total": 0
  }
}
```

### 2. 检查数据库是否创建

```bash
ls ~/.claude-mem/claude-mem.db
```

**期望结果：** 文件存在

### 3. 检查钩子是否注册

```bash
cat ~/.claude/settings.json | python3 -m json.tool | grep -A5 "hooks"
```

**期望结果：** 看到 `PreToolUse`、`PostToolUse`、`Stop` 等钩子配置

### 4. 打开 Web UI

浏览器访问 http://localhost:37777

**期望结果：** 显示 claude-mem 控制面板，包含实时记忆流

---

## 语言模式配置

安装完成后，可以配置工作语言和行为模式。编辑 `~/.claude-mem/settings.json`：

```json
{
  "CLAUDE_MEM_MODE": "code--zh"
}
```

可用模式：

| 模式 | 说明 |
|------|------|
| `code` | 默认英文模式 |
| `code--zh` | 简体中文模式（已内置，无需额外安装） |
| `code--ja` | 日文模式 |

查看所有可用模式：

```bash
ls ~/.claude/plugins/marketplaces/thedotmack/plugin/modes/
```

更改模式后，需要**重启 Claude Code** 生效。

---

## 完成确认检查清单

- [ ] `node --version` 输出 >= 18
- [ ] `npx claude-mem install` 完成无报错
- [ ] `curl http://127.0.0.1:37777/api/health` 返回 `"status": "healthy"`
- [ ] `~/.claude-mem/claude-mem.db` 文件存在
- [ ] Claude Code 已重启
- [ ] （可选）Web UI http://localhost:37777 可访问
- [ ] （可选）语言模式已按需配置

---

## 下一步

安装完成后，继续阅读：
- [快速开始](02-quickstart.md) — 了解第一次会话后如何查看捕获的记忆，以及如何搜索历史
- [高级用法](03-advanced-usage.md) — 多机同步、生产环境调优
