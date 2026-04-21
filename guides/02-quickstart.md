# 快速开始

## 适用场景

本指南适用于：
- 已完成安装，想了解 claude-mem 如何工作
- 想学习如何用自然语言搜索历史记忆
- 想调整上下文注入行为获得更好体验

---

## 理解自动工作流

claude-mem 完全自动运行，无需手动干预。以下是一次完整会话的生命周期：

```
Claude Code 启动
  │
  ├── SessionStart 钩子
  │     └── 启动 Worker + 注入相关上下文（基于语义相似度）
  │
  ├── UserPromptSubmit 钩子（每次发送 Prompt）
  │     └── 注册会话 + 启动 SDK Agent + 语义搜索注入
  │
  ├── PostToolUse 钩子（每次工具调用后）
  │     └── 捕获观察 → 入队 Worker → AI 压缩 → 存储 SQLite + ChromaDB
  │
  └── Stop / SessionEnd 钩子
        └── 生成会话摘要 + 完成会话
```

**第一次使用时：** 第一个会话结束后才会有记忆被存储，第二次会话开始时才会有上下文注入。

---

## 查看捕获的记忆

### 方式一：Web UI（推荐）

浏览器访问 http://localhost:37777，可以看到：
- 实时观察流
- 会话历史
- 搜索界面
- 版本切换（Beta 通道）

### 方式二：数据库直接查询

> **AI 可自动执行**

```bash
# 查看总体统计
sqlite3 ~/.claude-mem/claude-mem.db "
  SELECT 'observations' as metric, COUNT(*) as value FROM observations
  UNION ALL SELECT 'summaries', COUNT(*) FROM session_summaries
  UNION ALL SELECT 'pending', COUNT(*) FROM pending_messages WHERE status='pending'
  UNION ALL SELECT 'active_sessions', COUNT(*) FROM sdk_sessions WHERE status='active';
"

# 查看最近 10 条观察
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT title, type, created_at FROM observations ORDER BY created_at DESC LIMIT 10;"

# 查看最近会话摘要
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT learned, created_at FROM session_summaries ORDER BY created_at DESC LIMIT 3;"
```

---

## 使用 MCP 搜索工具

claude-mem 提供 3 个 MCP 工具，遵循**3 层渐进式检索**模式，节省约 10 倍 Token：

### 第 1 层：搜索索引

```typescript
// 获取紧凑索引（每条结果约 50-100 tokens）
search(query="authentication bug", type="bugfix", limit=10)
```

返回观察 ID 列表和简短摘要，快速定位相关记录。

### 第 2 层：时间线上下文

```typescript
// 查看特定观察周围发生了什么
timeline(observation_id=123, window=5)
```

了解某次修复前后的完整上下文。

### 第 3 层：获取完整详情

```typescript
// 只获取过滤后相关 ID 的完整内容（每条约 500-1000 tokens）
get_observations(ids=[123, 456])
```

**最佳实践：** 始终先 `search` → 筛选 ID → 再 `get_observations`，避免一次加载所有内容。

### 实际搜索示例

```typescript
// 搜索上周的数据库修复
search(query="database migration fix", type="bugfix", limit=5)

// 搜索特定时间段的认证工作
search(query="JWT authentication", date_from="2026-04-01", date_to="2026-04-15")

// 获取项目相关的所有 API 设计决策
search(query="API endpoint design", project="my-app", limit=10)
```

---

## 使用 mem-search Skill 搜索

在 Claude Code 中可以直接用自然语言触发搜索：

```
/mem-search 上周我们如何解决端口冲突问题？
```

```
/mem-search 认证模块的架构决策
```

---

## 隐私控制

对于不想被记忆系统捕获的敏感内容，使用 `<private>` 标签：

```
<private>
API_KEY=sk-xxxxx
这段内容不会被 claude-mem 存储
</private>
```

---

## 基础配置调优

编辑 `~/.claude-mem/settings.json`：

```json
{
  "CLAUDE_MEM_MODE": "code--zh",
  "CLAUDE_MEM_SEMANTIC_INJECT": true,
  "CLAUDE_MEM_SEMANTIC_INJECT_LIMIT": 5,
  "CLAUDE_MEM_MAX_CONCURRENT_AGENTS": 3,
  "CLAUDE_MEM_TIER_ROUTING_ENABLED": true
}
```

| 配置项 | 默认值 | 推荐值 | 说明 |
|--------|--------|--------|------|
| `CLAUDE_MEM_SEMANTIC_INJECT` | true | true | 语义注入（相关性优于时间顺序） |
| `CLAUDE_MEM_SEMANTIC_INJECT_LIMIT` | 5 | 5 | 每次注入的观察数量上限 |
| `CLAUDE_MEM_MAX_CONCURRENT_AGENTS` | 2 | 3 | 并发 SDK Agent 数量 |
| `CLAUDE_MEM_TIER_ROUTING_ENABLED` | true | true | 分层路由，约节省 52% 成本 |

更改配置后重启 Claude Code 生效。

---

## 完成确认检查清单

- [ ] 完成至少一次 Claude Code 会话（产生记忆数据）
- [ ] 数据库中 `observations` 表有记录
- [ ] Web UI 可以看到会话历史
- [ ] 成功用 MCP `search` 工具查询到相关记忆
- [ ] 了解 `<private>` 标签的使用方式

---

## 下一步

- [高级用法](03-advanced-usage.md) — 多机同步、生产环境健康监控、Beta 功能
- [故障排查](../troubleshooting.md) — 遇到问题时参考
