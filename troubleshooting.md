# 故障排查

## 安装问题

---

### 问题 1：npm 未找到，无法执行 npx

**难度：** 低

**症状：**
```
npm : The term 'npm' is not recognized as the name of a cmdlet
```

或：
```
command not found: npx
```

**常见原因：**
- Node.js 未安装（概率 60%）
- Node.js 已安装但未加入 PATH（概率 30%）
- PATH 更改后终端未重启（概率 10%）

**排查步骤：**
1. 检查 Node.js 是否存在：`which node` 或 `where node`
2. 检查版本：`node --version`

**解决方案：**

方案 A（未安装）：
```bash
# 从 https://nodejs.org 下载 LTS 版安装器
# 安装完成后重启终端
node --version  # 验证
```

方案 B（PATH 问题，macOS/Linux）：
```bash
# 找到 node 实际路径
which node || ls /usr/local/bin/node

# 加入 PATH（以 zsh 为例）
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

### 问题 2：安装完成后 Worker 未启动

**难度：** 中

**症状：**
```
Worker failed to start... Is port 37777 in use?
```
或 `curl http://127.0.0.1:37777/api/health` 连接被拒绝。

**常见原因：**
- 端口 37777 被其他进程占用（概率 40%）
- Bun 未正确安装（概率 30%）
- 两个 Claude Code 会话同时启动导致竞争（概率 20%）
- 权限问题（概率 10%）

**排查步骤：**

> **AI 可自动执行以下命令**

```bash
# 检查端口占用
lsof -i :37777

# 检查 Bun 是否安装
bun --version

# 查看 Worker 日志
cat ~/.claude-mem/logs/claude-mem-$(date +%Y-%m-%d).log | tail -50
```

**解决方案：**

```bash
# 方案 A：杀掉占用端口的进程
kill $(lsof -ti :37777)

# 方案 B：手动重装 Bun
curl -fsSL https://bun.sh/install | bash

# 方案 C：重新运行安装（会自动修复大多数问题）
npx claude-mem install
```

---

### 问题 3：钩子未注册，会话结束后没有记忆

**难度：** 中

**症状：** 多次会话后，数据库中 observations 仍为 0。

**常见原因：**
- 安装后未重启 Claude Code（概率 70%）
- settings.json 钩子配置被覆盖（概率 20%）
- 安装时权限不足（概率 10%）

**排查步骤：**
```bash
# 检查钩子配置
cat ~/.claude/settings.json | python3 -m json.tool | grep -A 20 "hooks"

# 检查数据库
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) FROM observations;"
```

**解决方案：**
1. 重启 Claude Code（解决 70% 的情况）
2. 如果钩子不在 settings.json 中，重新执行 `npx claude-mem install`

---

## 使用问题

---

### 问题 4：注入的上下文与当前任务无关

**难度：** 低

**症状：** 讨论认证功能时，Claude 被注入了 CSS 样式相关的历史记录。

**常见原因：**
- `CLAUDE_MEM_SEMANTIC_INJECT` 被设置为 false（概率 60%）
- 语义注入数量太少（概率 30%）
- ChromaDB 向量库未建立（安装刚完成，历史不足）（概率 10%）

**排查步骤：**
```bash
cat ~/.claude-mem/settings.json
```

**解决方案：**
```json
// ~/.claude-mem/settings.json
{
  "CLAUDE_MEM_SEMANTIC_INJECT": true,
  "CLAUDE_MEM_SEMANTIC_INJECT_LIMIT": 5
}
```
重启 Claude Code 后生效。

---

### 问题 5：pending_messages 持续增长不消耗

**难度：** 中

**症状：**
```bash
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) FROM pending_messages WHERE status='pending';"
# 返回 > 10 且不断增长
```

**常见原因：**
- SDK Agent 处理卡住（概率 50%）
- Circuit-breaker 触发导致停止处理（概率 30%）
- Worker 内存不足（概率 20%）

**排查步骤：**
```bash
# 检查 circuit-breaker 是否触发
grep 'CIRCUIT\|circuit.breaker\|ABANDONED' \
  ~/.claude-mem/logs/claude-mem-$(date +%Y-%m-%d).log

# 检查 failed 消息
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) FROM pending_messages WHERE status='failed';"
```

**解决方案：**
```bash
# 方案 A：重启 Worker（通过 Web UI 或结束所有 Claude Code 会话再启动）
# 方案 B：手动清理 failed 消息
sqlite3 ~/.claude-mem/claude-mem.db \
  "UPDATE pending_messages SET status='pending' WHERE status='failed';"
```

---

### 问题 6：MCP 搜索工具返回空结果

**难度：** 中

**症状：** `search(query="...")` 返回空列表，但数据库中有观察数据。

**常见原因：**
- ChromaDB 向量同步未完成（新安装，概率 50%）
- Chroma MCP 进程未启动（概率 30%）
- 搜索词太具体，向量相似度过低（概率 20%）

**排查步骤：**
```bash
# 检查 SQLite 中是否有数据
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT COUNT(*) FROM observations;"

# 检查 ChromaDB 文件
ls -lh ~/.claude-mem/chroma.sqlite3

# 尝试更宽泛的搜索词（用英文关键词通常效果更好）
```

**解决方案：**
1. 等待 5-10 分钟让 ChromaDB 同步完成
2. 尝试使用 SQLite 全文搜索作为备选：
```bash
sqlite3 ~/.claude-mem/claude-mem.db \
  "SELECT title, narrative FROM observations WHERE title LIKE '%你的关键词%' LIMIT 5;"
```

---

## 网络/环境问题

---

### 问题 7：Chroma 批量同步失败

**难度：** 高

**症状（日志中出现）：**
```
[ERROR] Batch add failed... IDs already exist
```

**常见原因：**
- MCP 超时导致部分写入，重试时 ID 已存在（概率 90%）

**解决方案：**
此为已知问题（PR #1566 修复），通过 delete+add 协调模式解决。确保使用最新版 claude-mem：
```bash
npx claude-mem@latest install
```

---

### 问题 8：Summarize 错误循环

**难度：** 中

**症状（日志中重复出现）：**
```
[ERROR] Missing last_assistant_message
```

**常见原因：**
- 会话中没有 assistant 消息就触发了摘要（概率 95%）

**解决方案：**
此为已知问题（PR #1566 修复），升级到最新版即可：
```bash
npx claude-mem@latest install
```

---

### 问题 9：WAL 文件过大导致性能下降

**难度：** 低

**症状：** 数据库操作变慢，`~/.claude-mem/` 目录占用异常大。

**排查步骤：**
```bash
ls -lh ~/.claude-mem/claude-mem.db-wal
```

**解决方案：**
```bash
sqlite3 ~/.claude-mem/claude-mem.db "PRAGMA wal_checkpoint(TRUNCATE);"
```

---

## 通用诊断工具

当问题难以定位时，生成完整诊断报告：

```bash
cd ~/.claude/plugins/marketplaces/thedotmack && npm run bug-report
```

报告输出路径会在命令执行后显示。复制报告内容到 GitHub Issue 可以获得快速支持。

**GitHub Issues：** https://github.com/thedotmack/claude-mem/issues

**Discord 支持：** https://discord.com/invite/J4wttp9vDu
