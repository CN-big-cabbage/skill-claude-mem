# 高级用法

## 适用场景

本指南适用于：
- 在多台机器上使用 Claude Code 并希望共享记忆
- 需要监控 claude-mem 健康状态和性能
- 希望使用 Beta 通道试用实验性功能
- 需要在生产环境中精细调优 claude-mem 行为

---

## 多机记忆同步

如果你在家和公司分别使用 Claude Code，claude-mem-sync 可以在机器间同步记忆。

> **AI 可自动执行以下命令**

```bash
# 将本地记忆推送到远程
claude-mem-sync push my-remote-host

# 从远程拉取到本地
claude-mem-sync pull my-remote-host

# 双向同步
claude-mem-sync sync my-remote-host

# 比较本地与远程的观察数量差异
claude-mem-sync status my-remote-host
```

**去重机制：** 同步基于 `(created_at, title)` 去重，安全重复执行。

**推荐工作流：**
1. 工作机结束工作前：`claude-mem-sync push work-machine`
2. 家庭机开始工作前：`claude-mem-sync pull work-machine`

---

## 健康监控

### 快速健康检查

> **AI 可自动执行**

```bash
# Worker 健康状态
curl -s http://127.0.0.1:37777/api/health | python3 -m json.tool

# 完整数据库统计
sqlite3 ~/.claude-mem/claude-mem.db "
  SELECT 'observations' as metric, COUNT(*) as value FROM observations
  UNION ALL SELECT 'summaries', COUNT(*) FROM session_summaries
  UNION ALL SELECT 'pending', COUNT(*) FROM pending_messages WHERE status='pending'
  UNION ALL SELECT 'failed', COUNT(*) FROM pending_messages WHERE status='failed'
  UNION ALL SELECT 'active_sessions', COUNT(*) FROM sdk_sessions WHERE status='active';
"
```

### 健康指标参考

| 指标 | 健康 | 警告 | 行动 |
|------|------|------|------|
| `pending_messages (pending)` | 0-5 | > 10 | 检查 Worker 日志，可能需要重启 |
| `pending_messages (failed)` | 0 | > 0 增长 | Circuit-breaker 可能触发 |
| `sdk_sessions (active)` | 0-3 | > 5 卡住 | 孤儿会话，重启 Worker |
| WAL 文件大小 | < 10 MB | > 20 MB | 执行 WAL checkpoint |
| 每日错误数 | 0-2 | > 10 | 分析日志模式 |

### WAL 清理

```bash
sqlite3 ~/.claude-mem/claude-mem.db "PRAGMA wal_checkpoint(TRUNCATE);"
```

---

## 日志分析

> **AI 可自动执行**

```bash
# 统计每天的错误数量
grep '\[ERROR\]' ~/.claude-mem/logs/claude-mem-*.log | \
  sed 's/\[20[0-9][0-9]-[0-9][0-9]-/\n&/g' | \
  grep -oP '^\[20\d{2}-\d{2}-\d{2}' | sort | uniq -c

# 查找 circuit-breaker 触发记录
grep 'circuit\|Circuit\|ABANDONED\|abandoned' ~/.claude-mem/logs/claude-mem-*.log

# 查看今日 pending 消息处理状态
grep 'CLAIMED\|CONFIRMED\|FAILED\|ABANDONED' \
  ~/.claude-mem/logs/claude-mem-$(date +%Y-%m-%d).log | tail -20
```

---

## 生产环境调优

基于 23 天生产环境、3400+ 观察的实测推荐配置：

```json
{
  "CLAUDE_MEM_MAX_CONCURRENT_AGENTS": 3,
  "CLAUDE_MEM_SEMANTIC_INJECT": true,
  "CLAUDE_MEM_SEMANTIC_INJECT_LIMIT": 5,
  "CLAUDE_MEM_TIER_ROUTING_ENABLED": true
}
```

**存储增长预期：**

| 指标 | 每天 | 每月 |
|------|------|------|
| 观察数量 | ~120 | ~3,600 |
| 会话摘要 | ~40 | ~1,200 |
| SQLite 大小 | ~0.8 MB | ~24 MB |
| ChromaDB 大小 | ~4 MB | ~120 MB |

---

## Beta 功能：Endless Mode

Endless Mode 是仿生记忆架构，专为长时间会话设计。

**启用方式：**
1. 打开 Web UI：http://localhost:37777
2. 进入 Settings 页面
3. 切换到 Beta 通道
4. 重启 Claude Code

---

## 生成诊断报告

遇到问题时，生成诊断报告便于上报 GitHub Issue：

```bash
cd ~/.claude/plugins/marketplaces/thedotmack && npm run bug-report
```

报告包含系统信息、数据库状态、最近日志等，自动脱敏处理。

---

## 通知集成（OpenClaw）

通过 OpenClaw Gateway 安装后，可以将实时观察推送到：
- Telegram
- Discord
- Slack

详细配置参见 [OpenClaw 集成文档](https://docs.claude-mem.ai/openclaw-integration)。

---

## 引用历史观察

每条观察都有唯一 ID，可以直接引用：

```bash
# 通过 API 获取特定观察
curl http://localhost:37777/api/observation/123

# 在 Web UI 中浏览所有观察
# 访问 http://localhost:37777
```

---

## 完成确认检查清单

- [ ] 多机同步配置完成（如有需要）
- [ ] 已了解关键健康指标含义
- [ ] 已配置生产推荐参数
- [ ] 知道如何查看日志和生成诊断报告

---

## 相关链接

- [故障排查](../troubleshooting.md) — 常见问题解决
- [官方文档：Context Engineering](https://docs.claude-mem.ai/context-engineering) — 上下文优化原则
- [官方文档：Architecture](https://docs.claude-mem.ai/architecture/overview) — 系统架构深入
