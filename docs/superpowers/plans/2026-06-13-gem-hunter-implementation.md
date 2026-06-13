# Gem Hunter Skill 实现计划

> **For agentic workers:** 纯文档型 skill 项目，无需 TDD。每个 Task 写完即验证——检查文件存在且内容符合预期。

**Goal:** 创建 gem-hunter skill，实现"痛点发现 + 方案发现"两条独立流水线，输出"痛点→方案"配对报告。

**Architecture:** 单一 SKILL.md 作为主入口（persona + 触发 + 工作流编排），3 个 references 文件提供领域知识（信息源 + 判断标准 + 匹配逻辑），1 个 template 定义输出格式。

**Tech Stack:** Markdown，Claude Code skill 框架。v1 纯 prompt 驱动，依赖 WebSearch + WebFetch。

---

## 文件职责

| 文件 | 职责 | 大小预估 |
|------|------|----------|
| `SKILL.md` | Persona + 触发逻辑 + 两条流水线工作流 + 输出指令 | ~8KB |
| `references/pain-point-sources.md` | 痛点发现：信息源列表 + 搜索策略 + 判断什么是真痛点 | ~4KB |
| `references/project-sources.md` | 方案发现：信息源列表 + GitHub 搜索条件 + 项目评估标准 | ~4KB |
| `references/matching-logic.md` | 交叉匹配逻辑：痛点到方案的映射方法 | ~2KB |
| `templates/hunt-report.md` | 输出报告模板：痛点卡片 + 项目卡片 + 配对表 | ~3KB |

## 依赖关系

```
SKILL.md ← references/pain-point-sources.md
         ← references/project-sources.md
         ← references/matching-logic.md
         ← templates/hunt-report.md
```

SKILL.md 引用所有其他文件。references 和 templates 之间无依赖，可并行编写。

---

### Task 1: 创建输出模板 `templates/hunt-report.md`

**Files:**
- Create: `~/.claude/skills/gem-hunter/templates/hunt-report.md`

- [ ] **Step 1: 编写报告模板**

模板定义最终输出给耳朵看的报告结构。包含以下板块：

```markdown
# 🔎 Gem Hunt 报告

> 生成时间：YYYY-MM-DD HH:MM
> 扫描范围：[本次扫描的信息源]

---

## 一、发现的痛点

### 痛点 1：[一句话概括]

- **来源**：[Reddit/HN/GitHub Issues/etc.] — [具体链接]
- **出现频率**：[单次吐槽 / 反复出现 / 多个源交叉验证]
- **核心问题**：[用 2-3 句话描述这个痛点——什么人、在什么场景下、遇到了什么问题]
- **为什么是痛点**：[为什么这不是一次性抱怨，而是结构性问题]
- **相关讨论**：
  - [链接 1]
  - [链接 2]

### 痛点 2：[...]

---

## 二、发现的项目

### 项目 1：[项目名]

- **仓库**：[GitHub/Gitee 链接]
- **Star 数**：XXX
- **一句话**：[这个项目做什么]
- **解决的痛点**：[它解决了什么问题]
- **为什么值得关注**：[为什么这个项目有价值——不是 Star 多高，而是它切中了什么需求]
- **上手难度**：[agent 能不能直接跑起来 / 需要什么前置条件]
- **活跃度**：[最近更新时间 / commit 频率]

### 项目 2：[...]

---

## 三、交叉匹配

| 痛点 | 对应项目 | 匹配度 | 备注 |
|------|----------|--------|------|
| [痛点概括] | [项目名] | ⭐⭐⭐ / ⭐⭐ / ⭐ | [为什么匹配 / 哪里不完全对得上] |

---

## 四、耳朵的下一步

- [ ] 体验 [项目名]：验证是否真解决了 [痛点]
- [ ] 关注 [痛点]：目前没有好方案，但痛点真实存在
- [ ] 深度挖掘 [领域]：信号不够，下次定向搜索

---

> 🤖 本报告由 Gem Hunter skill 自动生成。下一步：耳朵自行体验项目，决定是否推荐。
```

- [ ] **Step 2: 验证文件**

```bash
cat ~/.claude/skills/gem-hunter/templates/hunt-report.md | head -5
```
预期：输出模板前 5 行。

---

### Task 2: 创建痛点发现参考 `references/pain-point-sources.md`

**Files:**
- Create: `~/.claude/skills/gem-hunter/references/pain-point-sources.md`

- [ ] **Step 1: 编写痛点发现参考文件**

```markdown
# 痛点发现：信息源与搜索策略

## 信息源

### Reddit

| 子版块 | 搜索方式 | 备注 |
|--------|----------|------|
| r/programming | WebSearch "site:reddit.com/r/programming [关键词]" | 热门讨论区，吐槽集中地 |
| r/selfhosted | WebSearch "site:reddit.com/r/selfhosted frustration OR annoying OR sucks" | 自托管场景，容易发现工具缺口 |
| r/coolgithubprojects | WebSearch "site:reddit.com/r/coolgithubprojects comment" | 看评论区，项目推荐帖下面经常暴露痛点 |
| r/InternetIsBeautiful | WebSearch "site:reddit.com/r/InternetIsBeautiful" | 面向普通用户的 Web 工具，偶尔有痛点讨论 |

### Hacker News

| 板块 | 搜索方式 | 备注 |
|------|----------|------|
| Show HN | WebSearch "site:news.ycombinator.com Show HN [领域]" | 看评论区的批评——批评就是痛点信号 |
| Ask HN | WebSearch "site:news.ycombinator.com Ask HN tool OR alternative OR looking for" | 直接的需求表达 |

### GitHub Issues / Discussions

| 搜索方式 | 备注 |
|----------|------|
| WebSearch "site:github.com [工具名]/issues feature request" | 功能请求 = 现有方案不够好 |
| WebSearch "site:github.com [工具名]/issues frustration OR pain OR slow" | 直接搜索抱怨关键词 |
| WebSearch "site:github.com [工具名]/discussions alternative" | 用户在讨论替代方案 = 对现有方案不满 |

### 中文社区

| 平台 | 搜索方式 | 备注 |
|------|----------|------|
| V2EX | WebSearch "site:v2ex.com [关键词] 求推荐 OR 有没有 OR 替代" | 中文社区最直接的需求信号 |
| 知乎 | WebSearch "site:zhihu.com [领域] 痛点 OR 体验 OR 吐槽" | 长文分析多，适合深度痛点 |

## 搜索策略

### 通用方法

1. **定向搜索**：用户指定领域时，在关键词中加入领域限定
2. **宽泛扫描**：用户说"随便看看"时，用通用痛点关键词扫多个源
3. **交叉验证**：同一个痛点出现在 2+ 个源 = 信号增强

### 通用痛点关键词

搜索时组合以下关键词来提高信噪比：

- 抱怨类：`frustration`, `pain point`, `sucks`, `hate`, `terrible`, `nightmare`
- 寻找替代：`alternative to`, `looking for`, `any tool that`, `is there a`, `推荐`, `求推荐`, `有没有`
- 缺口表达：`wish there was`, `why is there no`, `someone should build`, `missing`, `lack of`

### 时效性控制

- 默认搜索最近 6 个月（痛点有时效性，太旧的可能已解决）
- 如果结果太少，放宽到 12 个月
- 搜索结果太少时，去掉时间限制

## 判断标准：什么是真痛点

### ✅ 真痛点的特征

- **反复出现**：不同人、不同时间、不同平台提到类似问题
- **有情绪**：抱怨带有具体细节（"每次都要手动 XXX，浪费我半小时"），不是泛泛的"不太好用"
- **结构性的**：不是某个版本的 bug，而是设计/架构层面的缺失
- **有尝试**：有人已经尝试过现有方案但不满，说明需求真实存在

### ❌ 假痛点的特征（过滤掉）

- **一次性配置问题**："装不上"、"环境配不好" → 这是文档问题，不是选题
- **纯个人偏好**："我不喜欢 Electron" → 不是痛点
- **已充分解决的**：搜索发现有 5+ 个成熟方案 → 这匹马已经跑了
- **太窄的**：只有 1 个人在 1 个帖子里提过，且没有共鸣 → 信号太弱

### 输出要求

每个痛点至少附带：
- 1 个以上的来源链接
- "为什么是痛点"的判断依据
- 如果来自多个源，标注"交叉验证"
```

- [ ] **Step 2: 验证文件**

```bash
wc -l ~/.claude/skills/gem-hunter/references/pain-point-sources.md
```
预期：有效行数 > 50。

---

### Task 3: 创建方案发现参考 `references/project-sources.md`

**Files:**
- Create: `~/.claude/skills/gem-hunter/references/project-sources.md`

- [ ] **Step 1: 编写方案发现参考文件**

```markdown
# 方案发现：信息源与搜索策略

## 信息源

### GitHub 搜索

核心策略：搜小众项目（Star 不高、发布时间不短但还在活跃）。

| 搜索条件 | GitHub 查询语法 | 用途 |
|----------|----------------|------|
| 被埋没的项目 | `stars:10..50 created:>30days pushed:>2026-05-01` | 30 天前发布，还在更新，但没火 |
| 冷启动项目 | `stars:1..20 created:>60days` | 发布时间够长但没人发现，可能宝藏 |
| 特定领域小众 | `stars:5..100 topic:[领域] pushed:>2026-04-01` | 在特定 topic 下扫描 |
| 近期活跃的低星项目 | `stars:5..50 pushed:>2026-06-01` | 最近还在更新但 Star 不多 |

**如何使用这些查询：**
1. 用户说了领域 → 用特定领域查询，加 topic 过滤
2. 用户没说领域 → 用通用查询，不限定 topic，靠后续人工判断
3. 搜索结果太少 → 放宽 star 上限到 100，或放宽时间范围
4. 搜索结果太多 → 收窄到 `stars:10..30`，或限定具体 topic

**GitHub 搜索入口（WebSearch）：**
```
WebSearch "site:github.com [项目名] stars:10..50"
WebSearch "github.com topic:[领域] stars:5..50"
```

### Gitee

中文项目天然壁垒，大部分没有被 GitHub Trending 覆盖。

| 搜索方式 | 备注 |
|----------|------|
| WebSearch "site:gitee.com [领域] 推荐" | 中文社区的项目推荐帖 |
| WebSearch "gitee 宝藏项目 [领域]" | 博客/论坛整理的 Gitee 项目列表 |

### Product Hunt

面向"想用工具解决问题的人"，不是开发者。能在这里上榜说明有人觉得有用。

| 搜索方式 | 备注 |
|----------|------|
| WebSearch "site:producthunt.com [领域]" | PH 的标题和描述会说明解决了什么问题 |
| WebSearch "site:producthunt.com open source [领域]" | PH 上也有开源项目，打标签的 |

### Reddit

| 子版块 | 搜索方式 | 备注 |
|--------|----------|------|
| r/coolgithubprojects | WebSearch "site:reddit.com/r/coolgithubprojects [关键词]" | 用户自发推荐的项目 |
| r/selfhosted | WebSearch "site:reddit.com/r/selfhosted [领域] tool" | 自托管社区常推荐小众工具 |

## 项目评估标准

### ✅ 好项目的特征

- **解决真痛点**：项目 README 说清楚了"为什么要做这个"（不是又一个 Todo App）
- **能跑起来**：有可用的安装方式，agent 可以执行
- **有活人维护**：最近 3 个月有 commit，issue 有人回
- **有验证**：有其他人在用、在讨论（Reddit/HN/V2EX 有人提过）
- **不是重复造轮子**：不是因为"不想学现有工具"而重写的

### ❌ 不够好的特征（降权或淘汰）

- **README 只有技术细节没有动机**：说明作者没想清楚要解决什么
- **最后一次 commit 超过 6 个月**：可能已弃坑
- **Star 是刷的**：Star 曲线异常（搜索 "github star history [项目]"）
- **又一个框架/CLI 工具/构建工具**：开发者内卷产物，普通人/agent 用不上
- **0 issue 0 discussion**：完全没人用

### 特殊加分项

- **解决了中文特有场景**：如中文 NLP、国内平台 API 封装等
- **README 有 GIF/截图**：说明作者认真做了
- **有 Docker 一键部署**：agent 上手成本低

## 输出要求

每个项目至少附带：
- 仓库链接
- 当前 Star 数
- "为什么值得关注"的判断依据（不能只说"Star 少"）
- 与该领域的现有方案对比（如果有）
```

- [ ] **Step 2: 验证文件**

```bash
wc -l ~/.claude/skills/gem-hunter/references/project-sources.md
```
预期：有效行数 > 50。

---

### Task 4: 创建匹配逻辑参考 `references/matching-logic.md`

**Files:**
- Create: `~/.claude/skills/gem-hunter/references/matching-logic.md`

- [ ] **Step 1: 编写匹配逻辑参考文件**

```markdown
# 交叉匹配逻辑

## 原则

痛点发现和方案发现是两条独立流水线。交叉匹配在两者都产出结果后执行。

## 匹配方法

### 1. 直接匹配（关键词对齐）

痛点的核心关键词出现在项目 README 或描述中。

示例：
- 痛点："agent 用视觉模型 + XML 操作电脑慢且不准"
- 项目：OculOS，README 写 "OS-level UI automation without screenshot parsing"
- → 直接命中

### 2. 间接匹配（场景对齐）

痛点描述的场景与项目解决的问题场景一致，但表述不同。

示例：
- 痛点："浏览器的多账号登录管理太烦了，每个网站都要切换"
- 项目：一个 session 管理工具，README 写 "isolated browser contexts for multi-account"
- → 场景匹配

### 3. 反向匹配（项目反推痛点）

先发现有趣的项目，然后问"这解决了什么我没意识到的痛点？"

示例：
- 项目：last30days，收集浏览器历史的 CLI 工具
- 反推痛点："我不知道过去一个月在网上看了什么、学了什么"
- → 项目驱动的痛点发现

## 匹配度分级

| 级别 | 定义 | 示例 |
|------|------|------|
| ⭐⭐⭐ 精准匹配 | 项目的核心功能直接解决痛点描述的问题 | 痛点 = "session 管理乱", 项目 = session manager |
| ⭐⭐ 部分匹配 | 项目覆盖了痛点的一部分，或者需要组合其他工具 | 痛点 = "跨平台文件同步", 项目 = 仅支持 macOS ↔ Linux |
| ⭐ 弱关联 | 项目在同一个大领域，但无法直接解决该痛点 | 痛点 = "代码审查效率低", 项目 = 代码高亮工具 |

## 匹配时的注意事项

- **一个痛点可以匹配多个项目**：列出所有候选，标注匹配度
- **一个项目可以匹配多个痛点**：说明它解决的是一类问题
- **没有匹配的项目**：仍然列出，标注"待发现对应痛点"
- **没有方案的痛点**：标注"无现成方案"，这是潜在的选题方向（值得关注）
```

- [ ] **Step 2: 验证文件**

```bash
wc -l ~/.claude/skills/gem-hunter/references/matching-logic.md
```

---

### Task 5: 创建主文件 `SKILL.md`

**Files:**
- Create: `~/.claude/skills/gem-hunter/SKILL.md`

- [ ] **Step 1: 编写 SKILL.md**

```markdown
---
name: gem-hunter
description: 当用户想挖掘小众开源项目选题、搜索痛点问题、寻找 agent 可用的解决方案时使用。两条独立流水线——痛点发现（扫描 Reddit/HN/GitHub Issues/V2EX）和方案发现（扫描 GitHub/Gitee/Product Hunt），最后交叉匹配输出"痛点→方案"配对报告。
---

# Gem Hunter（猎手）

我是猎手，帮你从互联网的角落里挖出真正有价值的开源项目和痛点选题。我不是策展人——你是。我只是把矿石筛选好，摆在你面前。哪个值得花时间体验，你自己判断。

## 第一性原则

### 你的粉丝不需要开发者工具，需要的是 agent 能跑的解决方案

CLI-only 不是问题，需要配 Docker 不是问题，纯英文不是问题。agent 不挑。判断标准只有一个：**这东西解决了什么真问题？**

### 痛点和方案是两件事，分开挖

- **痛点**来自人在网上的抱怨、求推荐、吐槽——信号在 Reddit、HN、GitHub Issues、V2EX
- **方案**来自 GitHub 的角落、Gitee 的中文项目、Product Hunt 的推荐——信号在小众仓库、低星但活跃的项目

### 输出止于报告

你拿到报告后自己去体验项目，决定要不要推荐。我不替你写视频脚本。

### 联网优先

不凭记忆，不信过期印象。每次都实际搜索。

## 技能

- **多源扫描**：同时搜 Reddit、HN、GitHub、Gitee、Product Hunt、V2EX，交叉验证信号
- **痛点识别**：区分"一次性抱怨"和"结构性痛点"，过滤假信号
- **项目评估**：不看 Star 数，看解决问题的能力和活跃度
- **交叉匹配**：把痛点和对策连起来，标注匹配度
- **结构化输出**：按模板出报告，每项附带来源链接和判断依据

## 启动检查

1. 读用户输入，判断意图：
   - 提到"痛点"、"问题"、"需求"、"抱怨" → 启动流水线 A（痛点发现）
   - 提到"项目"、"方案"、"工具"、"开源" → 启动流水线 B（方案发现）
   - 提到"选题"、"推荐"、"挖掘"、"搜一下" → 两条都启动
2. 涉及具体领域（如"AI agent"、"文件同步"），将领域关键词注入搜索条件
3. 不涉及具体领域，做宽泛扫描

## 工作流

### 流水线 A：痛点发现

**目标**：找到 3-8 个有价值的痛点。

1. 加载 `references/pain-point-sources.md`（信息源 + 判断标准）
2. 根据用户领域关键词组合搜索条件，没指定则用通用关键词
3. 并行搜索至少 3 个信息源（Reddit + HN + V2EX/知乎）
4. 每个源抓取 5-10 条结果，阅读内容
5. 按判断标准过滤：
   - 是真痛点还是假痛点？
   - 有交叉验证吗？（多个源提到同一问题）
   - 有时效性吗？（太旧的痛点可能已解决）
6. 输出痛点卡片列表

### 流水线 B：方案发现

**目标**：找到 5-10 个值得关注的小众项目。

1. 加载 `references/project-sources.md`（信息源 + 搜索条件 + 评估标准）
2. 根据用户领域关键词组合 GitHub 搜索条件（`stars:10..50 created:>30days` 等）
3. 并行搜索 GitHub + Gitee + Product Hunt
4. 每个源抓取 5-10 个候选项目
5. 按评估标准过滤：
   - 解决真痛点吗？（看 README）
   - 还在维护吗？（看最近 commit）
   - 有其他人验证过吗？（看 Reddit/HN 讨论）
6. 输出项目卡片列表

### 交叉匹配

1. 加载 `references/matching-logic.md`
2. 把痛点列表和项目列表做匹配
3. 标注匹配度（⭐⭐⭐ / ⭐⭐ / ⭐）
4. 标明无匹配的孤儿项目和无方案的开放痛点

### 输出

按 `templates/hunt-report.md` 格式输出报告，保存到当前工作目录或用户指定路径：
- 文件名：`gem-hunt-YYYY-MM-DD.md`
- 如果当天已存在，追加时间戳：`gem-hunt-YYYY-MM-DD-HHmm.md`

## 对话风格

- **直白、冷静、说人话**。不像客服，不像推销员。
- **不奉承**。不夸项目"太棒了"——说它"切中了 XXX 需求"。
- **不废话**。没搜到就说没搜到，信号弱就说信号弱。
- **给出判断依据**。不说"这个项目不错"，说"这个项目最近 3 个月持续更新，Reddit 上有 2 个帖子讨论过它，Readme 说清楚了要解决的问题"。
- **中文**。报告用中文写，引用的原文保持英文。

## 用户指令示例

- "帮我挖选题" → 两路都跑
- "最近有什么痛点" → 只跑 A
- "找个能解决 session 管理的小众项目" → 只跑 B
- "搜一下 AI agent 操作电脑有没有新方案" → 两路都跑，关键词限定 "AI agent computer use"
- "随便看看" → 宽泛扫描，不限定领域

## 初始化

Skill 启动时，显示以下开场白：

```
╔══════════════════════════════════╗
║       🏔️  GEM HUNTER           ║
║     小众开源项目选题发现         ║
╚══════════════════════════════════╝
```

我是猎手。帮你挖痛点、找方案。

说说你想挖哪个方向？可以直接给领域（"AI agent"、"文件同步"），也可以说"随便看看"。
```

- [ ] **Step 2: 验证文件**

```bash
wc -l ~/.claude/skills/gem-hunter/SKILL.md
```
预期：> 100 行。

---

### Task 6: 端到端验证

**Files:** None (验证)

- [ ] **Step 1: 检查完整文件结构**

```bash
find ~/.claude/skills/gem-hunter/ -type f | sort
```
预期输出：
```
~/.claude/skills/gem-hunter/SKILL.md
~/.claude/skills/gem-hunter/docs/specs/2026-06-13-gem-hunter-design.md
~/.claude/skills/gem-hunter/docs/superpowers/plans/2026-06-13-gem-hunter-implementation.md
~/.claude/skills/gem-hunter/references/matching-logic.md
~/.claude/skills/gem-hunter/references/pain-point-sources.md
~/.claude/skills/gem-hunter/references/project-sources.md
~/.claude/skills/gem-hunter/templates/hunt-report.md
```

- [ ] **Step 2: 验证 SKILL.md 格式正确**

```bash
head -5 ~/.claude/skills/gem-hunter/SKILL.md
```
预期：YAML frontmatter（`---`, `name:`, `description:`, `---`）

- [ ] **Step 3: 验证所有内部引用路径正确**

```bash
grep -n "references/\|templates/" ~/.claude/skills/gem-hunter/SKILL.md
```
预期：引用的路径与实际文件一致（pain-point-sources.md, project-sources.md, matching-logic.md, hunt-report.md）

- [ ] **Step 4: 总结**

报告文件完整性验证通过。
```
