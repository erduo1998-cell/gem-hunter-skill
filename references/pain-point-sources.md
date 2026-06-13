# 痛点发现：信息源与搜索策略

## 数据采集核心原则

**不用搜索引擎搜论坛内容。** 搜索引擎按 SEO 权重排序，不按讨论热度。真正有价值的讨论——Reddit 上一个吐槽帖下面 47 条评论——搜索引擎几乎不会排在前面。

**直接调平台 API，拿到一手讨论数据。** 每条结果带互动量（点赞、回复数），可以判断是真热点还是僵尸帖。

---

## 信息源

### Reddit（JSON API，免费，无需认证）

Reddit 提供免费 JSON API，只需加 User-Agent。返回按热度/评论数排序的帖子，带 score（赞数）和 num_comments（评论数）。

⚠️ **Reddit 的 shreddit 反爬系统可能封锁代理 IP，`.json` API 返回 HTML 空壳而非 JSON。** 遇到此情况自动降级为 WebSearch `site:reddit.com` 兜底，标注"Reddit API 被封锁，搜索引擎中转"。

**API 端点：**
```
https://www.reddit.com/r/{subreddit}/search.json?q={关键词}&sort=comments&restrict_sr=on&t=year&limit=25
```

**子版块与搜索关键词：**

| 子版块 | 搜索关键词 | 备注 |
|--------|----------|------|
| r/selfhosted | `frustrating OR annoying OR sucks OR alternative` | 自托管场景，吐槽集中地，痛点信号最强 |
| r/programming | `hate OR terrible OR nightmare OR looking for` | 热门讨论区 |
| r/coolgithubprojects | 项目推荐帖下的评论区 | 看评论有谁在说"这东西还缺 XXX" |
| r/InternetIsBeautiful | `wish there was OR why is there no` | 面向普通用户的 Web 工具需求 |
| r/foss | `alternative to OR looking for` | 自由开源软件社区的替代方案讨论 |
| r/privacy | `tool OR alternative OR self hosted` | 隐私工具需求 |

**调用方式：**
```bash
# 加载代理配置
HTTPS_PROXY=$(grep '^HTTPS_PROXY=' ~/.config/last30days/.env | cut -d= -f2-)

curl -s -x "$HTTPS_PROXY" -H "User-Agent: gem-hunter/1.0" \
  "https://www.reddit.com/r/selfhosted/search.json?q=frustrating+OR+annoying+OR+sucks&sort=comments&restrict_sr=on&t=year&limit=25" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for p in data['data']['children']:
    d = p['data']
    print(f'{d[\"title\"]} | {d[\"score\"]}pts | {d[\"num_comments\"]}cmt | r/{d[\"subreddit\"]}')
"

# 如果 curl 返回 HTML（<body class=theme-beta>）而非 JSON → API 被封锁
# 降级：WebSearch "site:reddit.com {关键词}"，标注"Reddit API 被封锁，搜索引擎中转"
```

**信号判断：**
- `score`（赞数）和 `num_comments`（评论数）越高，痛点信号越强
- 看完帖子标题后，点进评论看内容——"我也遇到过"、"目前最好的方案是 XXX 但还不行"→ 强信号
- `sort=comments` 优先显示讨论热烈的帖子
- 时间控制：`t=year` 可改为 `t=month` 搜最近一个月

### Hacker News（Algolia API，免费）

Algolia 提供的 HN 搜索 API，可搜帖子和评论，按点数/日期排序。

**API 端点：**
```
https://hn.algolia.com/api/v1/search?query={关键词}&tags=comment&hitsPerPage=20
https://hn.algolia.com/api/v1/search?query={关键词}&tags=story&hitsPerPage=20
```

**搜索场景：**

| 场景 | tags | 备注 |
|------|------|------|
| 评论中找痛点 | `tags=comment` | 评论区比帖子本身更有信号 |
| Show HN 帖子 | `tags=show_hn` | 看评论区的批评——批评就是痛点信号 |
| Ask HN | `tags=ask_hn` | 直接的需求表达（"Ask HN: Is there a tool that..."） |
| 全文搜索 | 不加 tags 限制 | 广撒网 |

**调用方式：**
```bash
# 加载代理配置
HTTPS_PROXY=$(grep '^HTTPS_PROXY=' ~/.config/last30days/.env | cut -d= -f2-)

SINCE=$(date -v-6m +%s)
curl -s -x "$HTTPS_PROXY" "https://hn.algolia.com/api/v1/search?query=tool+alternative+OR+looking+for&tags=comment&hitsPerPage=20&numericFilters=created_at_i>$SINCE" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for h in data['hits']:
    title = h.get('story_title') or h.get('title', '(no title)')
    print(f'{title} | {h.get(\"points\", 0)}pts | {h[\"author\"]}')
"
```

**信号判断：**
- `points` 高的评论 = 更多人共鸣
- Ask HN 里的"Ask HN: Is there a tool that..." → 直接痛点，最高优先级
- Show HN 评论区有人说"nice but I wish it could..." → 功能缺口信号
- 搜索结果中同一话题出现多次 → 交叉验证

### X/Twitter（Bird 引擎，需认证）

**这是痛点挖掘最关键的源。** X 上用户实时吐槽工具的密度远超任何论坛。gem-hunter 复用 last30days 的 vendored Bird 搜索引擎（`@steipete/bird`），直接调 Twitter GraphQL API，每条推文带点赞/转发/回复数。

**前置条件：**
- 需要有 AUTH_TOKEN 和 CT0（从浏览器 cookie 提取），存储在 `~/.config/last30days/.env`
- 需要 Node.js 可用
- Bird 脚本位置：`~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs`

**认证加载：**
```bash
source <(grep -E '^(AUTH_TOKEN|CT0)=' ~/.config/last30days/.env)
```

**痛点搜索关键词组合：**

| 关键词 | 用途 | 示例查询 |
|--------|------|---------|
| `frustrated with` | 直接抱怨 | `frustrated with {领域} since:2026-01-01` |
| `looking for alternative` | 寻找替代方案 | `looking for alternative {工具名} since:2026-01-01` |
| `wish there was` | 表达工具缺口 | `wish there was a tool that since:2026-01-01` |
| `any tool that` | 直接求助 | `any tool that can {功能} since:2026-01-01` |
| `hate {工具} but` | 对现有方案不满 | `hate github but since:2026-01-01` |
| `why is there no` | 缺口表达 | `why is there no {功能} since:2026-01-01` |
| `someone should build` | 创业/项目点子 | `someone should build {领域} since:2026-01-01` |

**调用方式：**
```bash
source <(grep -E '^(AUTH_TOKEN|CT0)=' ~/.config/last30days/.env)
SINCE=$(date -v-6m +%Y-%m-%d)
BIRD=~/.claude/skills/last30days/scripts/lib/vendor/bird-search/bird-search.mjs

# 直接抱怨 + 找替代方案
node "$BIRD" \
  "frustrated with OR looking for alternative since:$SINCE" \
  --count 30 --json > /tmp/x-pain.json

# 工具缺口 + 求助
node "$BIRD" \
  "wish there was a tool OR someone should build since:$SINCE" \
  --count 30 --json > /tmp/x-gap.json
```

**输出格式（JSON）：** 每条推文包含 text、url、author_handle、date、engagement（likes/reposts/replies/quotes）。

**信号判断：**
- Likes > 50 且 Replies > 10 → 强信号，多人共鸣
- 多条互不相关的推文提到同一个痛点 → 交叉验证，极强信号
- 推文下有人回复"same"、"I need this too" → 痛点确认
- 单条推文 like 很多但少人回复 → 可能有共鸣但没讨论起来，标注"中等信号"
- 只有 1 个人提过、几乎没有互动 → 标注"信号弱，观察"

**如果没有 X auth：**
- 降级用 WebSearch `site:x.com {关键词}` 兜底
- 明确标注"X 未认证，搜索引擎中转，信号偏弱"
- 提示用户：配置 AUTH_TOKEN/CT0 可大幅提升 X 搜索质量（运行 last30days 的 setup wizard 即可提取浏览器 cookie）

### GitHub Issues / Discussions（GitHub API）

**直接搜 Issues 内容，不用搜索引擎中转：**

| 搜索方式 | 工具 | 备注 |
|----------|------|------|
| Fossick MCP `search_code` | `search_code("frustrating OR pain point OR wish there was")` | 搜代码库中的讨论 |
| `gh search issues` | `gh search issues "frustrating" --limit 20` | 需要 gh CLI 已安装 |
| GitHub Discussion 搜索 | `gh search issues "alternative to" --type=discussion` | 替代方案探讨 |

**如果没有 gh CLI：** 用 Fossick MCP 的 `search_code` 搜全 GitHub 的 issue/discussion 内容。如果 Fossick 也不行，最后用 WebSearch 兜底，标注"搜索引擎中转，信号可能偏弱"。

**信号判断：**
- Issue 下的 👍 反应数 = 需求强度
- 多个不相关的 repo 出现类似 issue → 交叉验证，强信号
- Issue 标题带"frustrating"、"pain"、"slow" 的是直接抱怨

### 中文社区

#### 小红书（WebSearch 兜底）

⚠️ **MCP API 未配置（xiaohongshu-mcp Chrome 启动兼容问题）。当前走 WebSearch 中转。**

小红书是中文互联网最大的消费决策平台，痛点信号密度高。"求推荐"、"安利"、"避雷" 类笔记的真实现象讨论非常适合 gem-hunter 的选题发现。

| 搜索方式 | 备注 |
|----------|------|
| WebSearch `site:xiaohongshu.com {关键词} 推荐 OR 好用 OR 工具` | 正向推荐帖 |
| WebSearch `site:xiaohongshu.com {关键词} 求推荐 OR 安利` | 需求帖 |
| WebSearch `site:xiaohongshu.com {关键词} 避雷 OR 难用 OR 踩坑` | 吐槽帖 |

**信号判断：**
- 多条不同作者的笔记不约而同提到同一工具/痛点 → 交叉验证 ✅
- 单篇高赞笔记 → 中等信号（小红书算法推流可能放大单一内容）
- 笔记互动量（点赞+收藏）高但来自 KOL → 降权（KOL 推荐不等于真实需求）
- 普通用户（粉丝 < 1000）的真诚推荐 → 强信号
- 标注 ⚠️"MCP API 未配置，搜索引擎中转，信号可能偏弱"

**后续升级：** 待 xiaohongshu-mcp 可用后改为 API 直连（`POST /api/v1/feeds/search`），可获取真实互动数据而非搜索引擎片段。

#### V2EX（WebSearch 兜底）

V2EX 没有公开 API，只能通过 WebSearch。

| 搜索方式 | 备注 |
|----------|------|
| WebSearch `site:v2ex.com {关键词} 求推荐 OR 有没有 OR 替代` | 中文社区最直接的痛点信号 |
| WebSearch `site:v2ex.com 吐槽 OR 不好用 OR 难用 {领域}` | 抱怨帖 |

**⚠️ 标注规则：** 所有 V2EX 结果必须标注"搜索引擎中转，信号可能偏弱"。

---

## 搜索策略

### 通用方法

1. **定向搜索**：用户指定领域时，在关键词中加入领域限定
2. **宽泛扫描**：用户说"随便看看"时，用通用痛点关键词扫多个源
3. **交叉验证**：同一个痛点出现在 2+ 个源 = 信号增强
4. **API 优先**：X（Bird）、Reddit JSON API、HN Algolia、GitHub API 走 API 直连；V2EX 走 WebSearch 兜底

### 搜索范围

每个源搜 1-2 组关键词，每组 20-30 条结果，总候选控制在 80-150 条。

### 各源搜索关键词速查

| 源 | 推荐关键词组合 |
|----|-------------|
| X (Bird) | `frustrated with OR looking for alternative` / `wish there was a tool OR someone should build` |
| Reddit | `frustrating OR annoying OR sucks OR alternative` / `hate OR terrible OR nightmare` |
| HN | `tool OR alternative OR looking for` / `is there a OR wish there was` |
| GitHub | `frustrating OR pain point OR wish there was` |
| 小红书 | `推荐 OR 好用 OR 工具` / `求推荐 OR 安利` / `避雷 OR 难用 OR 踩坑` |
| V2EX | `求推荐 OR 有没有 OR 替代` / `吐槽 OR 不好用 OR 难用` |

### 时效性控制

- 默认搜索最近 6 个月（痛点有时效性，太旧的可能已解决）
- 如果结果太少，放宽到 12 个月
- X Bird 使用 `since:YYYY-MM-DD` 控制
- Reddit API 使用 `t=year` 或 `t=month`
- HN Algolia 使用 `numericFilters=created_at_i>{timestamp}`
- V2EX WebSearch 搜索时加时间限定词"2026"

### 信息源优先级

**第一梯队（API 直连，信号最干净）：**

| 优先级 | 源 | 原因 |
|--------|-----|------|
| 🔴 最高 | X/Twitter（Bird） | 实时吐槽密度最高，互动数据完整 |
| 🟠 高 | Reddit JSON API | 讨论深度最高，长文分析多 |
| 🟡 中高 | HN Algolia API | 技术社区信号，评论质量高 |
| 🟢 中 | GitHub Issues | 功能缺口信号，有 👍 反应数 |

**第二梯队（搜索引擎中转，⚠️ 信号可能偏弱）：**
- V2EX（WebSearch）
- 小红书（WebSearch，MCP API 未配置）

### 不使用的源

| 源 | 淘汰原因 |
|----|---------|
| 知乎 | 营销/AI 生成内容泛滥，信噪比低到不可用 |
| Product Hunt | 产品发布平台，标题和描述是创始人写的广告语，不是用户讨论 |
| 行业分析媒体（VentureBeat, a16z 等） | 产出的是"行业趋势"不是"个人痛点"，粒度不对 |
| 学术论文（NeurIPS 等） | 同上，不是选题素材 |

---

## 判断标准：什么是真痛点

### 最重要的闸门：粒度（必须先过这一关）

**一个好痛点的粒度：一个人 + 一个具体场景 + 一个工具能解决。**

- ✅ "每次在多台电脑间同步浏览器书签都要手动导出导入" → 具体、单人、有工具能解决
- ✅ "想给视频加字幕但不想上传到任何云服务" → 具体、单人、有工具能解决
- ❌ "AI Agent 上下文管理是行业瓶颈" → 这是行业趋势，不是选题
- ❌ "企业数据主权焦虑" → 这是市场报告标题，不是选题
- ❌ "开源社区平台老化" → 这是社会学议题，不是选题

**判定方法**：如果一个痛点需要引用 a16z 分析报告或行业论文来解释，它太大了。如果一个痛点你在论坛评论区看到有人吐槽、立刻觉得"对啊我也遇到过"——这才是你要的粒度。

### ✅ 真痛点的特征

- **反复出现**：不同人、不同时间、不同平台提到类似问题
- **有情绪**：抱怨带有具体细节（"每次都要手动 XXX，浪费我半小时"），不是泛泛的"不太好用"
- **可解决的**：不是一个行业的结构性问题，而是有具体工具/方案能解决的
- **有尝试**：有人已经尝试过现有方案但不满，说明需求真实存在
- **能一句话说清楚**：不需要背景知识、不需要行业术语，一个普通人看完就懂

### ❌ 假痛点的特征（过滤掉）

- **行业级趋势**："AI 行业缺少 XXX 基础设施"、"企业数字化转型的 XXX 困境" → 这是分析报告，不是选题
- **一次性配置问题**："装不上"、"环境配不好" → 这是文档问题，不是选题
- **纯个人偏好**："我不喜欢 Electron" → 不是痛点
- **已充分解决的**：搜索发现有 5+ 个成熟方案 → 这匹马已经跑了
- **太窄的**：只有 1 个人在 1 个帖子里提过，且没有共鸣 → 信号太弱
- **需要 200 字背景介绍才能理解的**：如果痛点需要解释"XX 行业的结构性问题"才能让人明白，淘汰

### 输出要求

每个痛点至少附带：
- 1 个以上的来源链接
- "为什么是痛点"的判断依据
- 互动数据（X 的 likes/replies、Reddit 的 score/comments、HN 的 points）
- 如果来自多个源，标注"交叉验证"
