---
name: claude-mem
description: Claude Code 跨会话持久化记忆插件，自动捕获工具使用观察、生成语义摘要并在未来会话中注入相关上下文，让 Claude 真正"记住"你的项目历史
version: 0.1.1
metadata:
  openclaw:
    requires:
      bins:
        - node
        - npx
    emoji: 🧠
    homepage: https://claude-mem.ai
---

# claude-mem — Claude Code 持久化记忆系统

claude-mem 是专为 Claude Code 打造的持久化记忆插件，通过 5 个生命周期钩子自动捕获每次编码会话中的工具使用观察，使用 Claude Agent SDK 进行 AI 压缩，并在下次会话开始时将最相关的上下文注入回来。它让 Claude 在跨会话场景下真正"记住"你的项目历史，无需手动粘贴上下文。

## 核心使用场景

- **跨会话连续开发**：明天打开 Claude Code 继续昨天的任务，Claude 自动了解昨日进度
- **大型项目上下文维护**：在拥有数百次历史观察的代码库中快速找到相关决策记录
- **多机器同步**：在家庭与工作机之间保持记忆一致（通过 claude-mem-sync）
- **语义搜索历史**：用自然语言查询"上周我们如何修复 auth bug"
- **隐私安全记录**：使用 `<private>` 标签排除敏感内容不被存储

## AI 辅助使用流程

1. **安装插件** — AI 执行 `npx claude-mem install`，自动配置钩子和 Worker 服务
2. **验证运行** — AI 检查 `http://localhost:37777/api/health` 确认 Worker 启动正常
3. **日常使用** — 完全自动：每次 Claude Code 会话的工具使用会被自动捕获和压缩
4. **查询记忆** — 使用 mem-search skill 或 MCP 工具以自然语言搜索历史观察
5. **调整配置** — AI 编辑 `~/.claude-mem/settings.json` 优化注入行为和模式
6. **健康监控** — AI 定期检查数据库统计和 Worker 健康状态

## 关键章节导航

- [安装指南](guides/01-installation.md) — 安装、验证和多 IDE 支持
- [快速开始](guides/02-quickstart.md) — 第一次会话、记忆搜索、配置调优
- [高级用法](guides/03-advanced-usage.md) — 多机同步、生产调优、MCP 工具深入
- [故障排查](troubleshooting.md) — Worker 启动失败、记忆不注入等常见问题

## AI 助手能力

使用本技能时，AI 可以：

- ✅ 执行 `npx claude-mem install` 完成一键安装和钩子注册
- ✅ 读取并修改 `~/.claude-mem/settings.json` 调整记忆行为
- ✅ 调用 `curl http://127.0.0.1:37777/api/health` 检查 Worker 健康状态
- ✅ 查询 SQLite 数据库获取观察数量、活跃会话等统计信息
- ✅ 使用 MCP `search` / `timeline` / `get_observations` 工具搜索历史记忆
- ✅ 分析日志文件定位 circuit-breaker 触发、pending 消息积压等问题
- ✅ 执行 `claude-mem-sync` 命令完成多机记忆同步
- ✅ 运行 `npm run bug-report` 生成诊断报告辅助问题上报

## 核心功能

- ✅ **5 个生命周期钩子** — SessionStart / UserPromptSubmit / PostToolUse / Stop / SessionEnd
- ✅ **Worker 守护进程** — 端口 37777，含 Web UI 和 10 个搜索端点
- ✅ **混合搜索** — SQLite 全文检索 + ChromaDB 向量语义搜索
- ✅ **3 层 MCP 工作流** — search → timeline → get_observations，节省约 10 倍 Token
- ✅ **语义注入** — 每次 Prompt 基于相似度选择最相关上下文（而非最近）
- ✅ **隐私控制** — `<private>` 标签排除敏感内容
- ✅ **多语言模式** — code / code--zh / code--ja
- ✅ **多机同步** — claude-mem-sync push/pull/sync
- ✅ **多 IDE 支持** — Claude Code / Gemini CLI / OpenCode
- ✅ **Beta 通道** — Endless Mode 等实验性功能

## 快速命令示例

```bash
# 安装插件（推荐方式）
npx claude-mem install

# 安装到 Gemini CLI
npx claude-mem install --ide gemini-cli

# 安装到 OpenCode
npx claude-mem install --ide opencode

# 检查 Worker 健康状态
curl -s http://127.0.0.1:37777/api/health | python3 -m json.tool

# 查看观察数量统计
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) as observations FROM observations;"

# 多机同步（推送本地到远程）
claude-mem-sync push my-remote-host

# 生成 bug 诊断报告
cd ~/.claude/plugins/marketplaces/thedotmack && npm run bug-report

# 查看所有可用 modes
ls ~/.claude/plugins/marketplaces/thedotmack/plugin/modes/
```

## 安装要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| Node.js | >= 18.0.0 | 必须 |
| Claude Code | 最新版 | 需要插件支持 |
| Bun | 任意 | 自动安装（若缺失） |
| uv | 任意 | 自动安装，用于向量搜索 |
| SQLite 3 | 任意 | 已捆绑 |

## 项目链接

- GitHub：https://github.com/thedotmack/claude-mem
- 官网：https://claude-mem.ai
- 文档：https://docs.claude-mem.ai
- Web UI：http://localhost:37777（安装后本地访问）
- Discord：https://discord.com/invite/J4wttp9vDu

