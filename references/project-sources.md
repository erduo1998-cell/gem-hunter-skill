# 方案发现：信息源与搜索策略

## 数据采集核心原则

**方案发现分两步：先找项目（GitHub/Fossick），再验证（Reddit/X/HN 社区讨论）。** 不用搜索引擎找项目——搜索引擎的排名机制会让已火的项目排前面，gem-hunter 要找的"被埋没的低星项目"根本排不上来。

---

## 信息源

### GitHub 搜索（Fossick MCP —— 核心发现工具）

核心策略：搜小众项目（Star 不高、发布时间不短但还在活跃）。**Gem Hunter 的核心目标是低星高质项目，Star 上限硬约束：500。超过 500 星的只在作为对比或特别说明时提及。**

**使用 Fossick MCP 的 `search_repos` 工具：**

| 搜索场景 | Fossick 参数 | 用途 |
|----------|-------------|------|
| 被埋没的项目 | `search_repos(stars_min=10, stars_max=50, created_after="30days", topics=["领域"])` | 30 天前发布，还在更新，但没火 |
| 冷启动项目 | `search_repos(stars_min=1, stars_max=20, created_after="60days")` | 发布时间够长但没人发现 |
| 特定领域小众 | `search_repos(stars_min=5, stars_max=100, topics=["领域"], pushed_after="YYYY-MM-DD")` | 在特定 topic 下扫描 |
| 近期活跃的低星项目 | `search_repos(stars_min=5, stars_max=50, pushed_after="YYYY-MM-DD")` | 最近还在更新但 Star 不多 |

**Fossick 深度评估工具链**（搜到候选后用这些深入了解）：

| 工具 | 用途 | 调用时机 |
|------|------|----------|
| `search_repos` | 多维度搜索 GitHub 仓库 | 第一步：发现候选 |
| `repo_tree` | 浏览仓库目录结构（无需 clone） | 看项目规模和组织 |
| `get_file` | 读取 README 或任意文件 | 判断项目是否说清楚了要解决的问题 |
| `list_tags` | 查看版本发布历史 | 判断维护活跃度 |
| `search_code` | 全 GitHub 公开文件搜索 | 找类似项目做对比 |

**如何使用这些查询：**
1. 用户说了领域 → 用 `search_repos` 加 `topics` 过滤
2. 用户没说领域 → 不限定 topics，靠后续人工判断
3. 搜索结果太少 → 放宽 `stars_max` 到 100，或放宽时间范围
4. 搜索结果太多 → 收窄到 `stars_max=30`，或限定具体 topics
5. 发现候选后 → 逐个用 `get_file` 读 README，用 `list_tags` 看活跃度

### Reddit（JSON API —— 社区验证）

**不搜项目本身，搜"有没有人在讨论这个项目"。** 一个项目即使 Star 少，如果 Reddit 上有人在真诚推荐它（不是作者自荐），就是强验证信号。

**子版块与搜索关键词：**

| 子版块 | 搜索关键词 | 备注 |
|--------|----------|------|
| r/coolgithubprojects | 项目名 | 用户自发推荐小众项目 |
| r/selfhosted | 项目名 + `tool OR recommendation` | 自托管社区推荐实用工具 |
| r/programming | 项目名 | 看有没有讨论 |
| r/InternetIsBeautiful | 项目名 | 面向普通用户的工具 |

**调用方式：**
```bash
# 加载代理配置
HTTPS_PROXY=$(grep '^HTTPS_PROXY=' ~/.config/last30days/.env | cut -d= -f2-)

curl -s -x "$HTTPS_PROXY" -H "User-Agent: gem-hunter/1.0" \
  "https://www.reddit.com/search.json?q={项目名}&sort=comments&t=year&limit=10" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for p in data['data']['children']:
    d = p['data']
    print(f'{d[\"title\"]} | {d[\"score\"]}pts | {d[\"num_comments\"]}cmt | r/{d[\"subreddit\"]}')
"

# 如果 curl 返回 HTML（<body class=theme-beta>）而非 JSON → API 被封锁
# 降级：WebSearch "site:reddit.com {项目名}"，标注"Reddit API 被封锁，搜索引擎中转"
```

**信号判断：**
- 有 2+ 个不同子版块在讨论该项目的 → 社区验证通过
- 帖子里有人在评论里说"been using this for months, it's great" → 强验证
- 只有作者本人发的帖子 → 不算验证
- 完全没有 Reddit 讨论 → 不是一票否决，但标注"无社区验证"（降权）

### X/Twitter（Bird 引擎 —— 社区验证）

**验证项目是否有人在讨论和推荐。** X 上的工具推荐密度高、时效性强，是最好的社区验证源之一。

**前置条件：**
- 需要 AUTH_TOKEN 和 CT0，存储在 `~/.config/last30days/.env`
- 需要 Node.js 可用
- Bird 脚本位置：`~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs`

**认证加载：**
```bash
source <(grep -E '^(AUTH_TOKEN|CT0)=' ~/.config/last30days/.env)
```

**验证搜索：**
```bash
source <(grep -E '^(AUTH_TOKEN|CT0)=' ~/.config/last30days/.env)
BIRD=~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs

# 搜项目名 + 推荐/使用信号
node "$BIRD" \
  "{项目名} recommend OR love OR using OR switched to since:2026-01-01" \
  --count 20 --json > /tmp/x-project.json
```

**信号判断：**
- 有 3+ 条独立推文讨论该项目的 → 社区验证通过
- 推文来自不同的人（不是项目作者转发的）→ 强验证
- 推文带"just discovered"、"can't believe this only has X stars" → 典型的宝藏项目信号
- 无人讨论 → 不否决，标注"X 无讨论"

**如果没有 X auth：** 跳过此步骤，标注"X 未认证，跳过社区验证"。

### Hacker News（Algolia API —— 社区验证）

**搜 HN 上有没有人提过这个项目。** HN 上的讨论质量高，一个 Show HN 帖子如果有几十个点，说明技术社区认可。

**调用方式：**
```bash
# 加载代理配置
HTTPS_PROXY=$(grep '^HTTPS_PROXY=' ~/.config/last30days/.env | cut -d= -f2-)

curl -s -x "$HTTPS_PROXY" "https://hn.algolia.com/api/v1/search?query={项目名}&tags=story&hitsPerPage=10" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for h in data['hits']:
    print(f'{h[\"title\"]} | {h.get(\"points\", 0)}pts | {h.get(\"num_comments\", 0)}cmt')
"
```

**信号判断：**
- Show HN 帖子 points > 10 → 技术社区认可
- 评论里有人说"tried it, works well" → 强验证
- 没人提过 → 不否决，标注"HN 无讨论"

---

## 项目评估标准

### ✅ 好项目的特征

- **解决真痛点**：项目 README 说清楚了"为什么要做这个"（不是又一个 Todo App）
- **聚焦单一问题**：README 第一句话就能让普通人理解它解决什么。如果一个项目需要先解释行业背景才能说明白它做什么——太大了
- **能跑起来**：有可用的安装方式，agent 可以执行
- **有活人维护**：最近 3 个月有 commit，issue 有人回
- **有社区验证**：Reddit/X/HN 上有真实用户讨论（不是作者自荐），这比 Star 数更可靠
- **不是重复造轮子**：不是因为"不想学现有工具"而重写的

### ❌ 不够好的特征（降权或淘汰）

- **README 只有技术细节没有动机**：说明作者没想清楚要解决什么
- **最后一次 commit 超过 6 个月**：可能已弃坑
- **Star 太高反而可疑**：Star > 10K 的不是"小众宝藏"，是"已经很火了"——偶尔可用但不是 gem-hunter 的核心目标
- **大厂开源项目**（微软/Google/Meta/阿里等）：天然有传播渠道，不是"被埋没的宝藏"
- **又一个框架/CLI 工具/构建工具**：开发者内卷产物，普通人/agent 用不上
- **基础设施/平台级项目**：如"重新定义 XX 工作流"、"统一 XX 基础设施"——太大，不是一个人用一个工具能解决的
- **0 issue 0 discussion 社区无讨论**：完全没人用

### 特殊加分项

- **解决了中文特有场景**：如中文 NLP、国内平台 API 封装等
- **README 有 GIF/截图**：说明作者认真做了
- **有 Docker 一键部署**：agent 上手成本低

---

## 输出要求

每个项目至少附带：
- 仓库链接
- 当前 Star 数
- "为什么值得关注"的判断依据（不能只说"Star 少"）
- 社区验证状态（Reddit/X/HN 是否有讨论，附带互动数据）
- 与该领域的现有方案对比（如果有）
