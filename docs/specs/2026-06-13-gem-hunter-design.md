# Gem Hunter Skill — 设计规格

> 2026-06-13 · v1.0

## 1. 定位

**挖痛点 + 找方案，止于"痛点→方案"配对报告。** 耳朵是策展人，skill 是侦察兵。

核心场景：耳朵是中文内容博主，分享 agent 可用的开源解决方案。粉丝需要的是"agent 能跑起来解决问题"的项目，不是给人用的 GUI 工具。GitHub Trending 信号偏向开发者偏好，与耳朵粉丝需求错位。

## 2. 两条独立流水线

```
流水线 A：痛点发现              流水线 B：方案发现
─────────────────────          ─────────────────────
扫描信息源                       扫描信息源
  ↓                               ↓
提取反复出现的真实痛点            按条件搜索小众项目
  ↓                               ↓
输出痛点卡片                     输出项目卡片
  ↓                               ↓
  └──────→ 交叉匹配 ←──────────┘
                ↓
        痛点→方案 配对报告
```

两条流水线独立运行，可单独触发也可同时跑。最后做交叉匹配。

## 3. 关键设计决策

### 3.1 不需要传统的"可用性过滤器"

旧方案的四道闸（UI/语言/门槛/场景）不适用。项目是给 agent 执行的，不是给人用的：

- CLI-only → 完全 OK
- 需要 Docker/数据库/API Key → OK，agent 能配
- 纯英文 → OK，agent 不挑语言
- 面向开发者场景 → OK，只要那是个真痛点

### 3.2 判断标准重新定义

**什么是好痛点**：反复出现、有人持续抱怨、现有方案被吐槽、不是一次性的配置问题而是结构性问题。

**什么是好项目**：解决了真实痛点、Star 不多但还在活跃、有其他用户验证过"这玩意确实有用"、不是又一个框架/库。

### 3.3 信息源

**痛点发现源**：
- Reddit：r/InternetIsBeautiful, r/selfhosted, r/coolgithubprojects, r/programming
- Hacker News "Show HN" comments + Ask HN
- GitHub Issues/Discussions（搜索 frustration 关键词）
- V2EX、知乎

**方案发现源**：
- GitHub 搜索：`stars:<50 created:>30days` 等小众条件
- Gitee（中文项目天然壁垒）
- Product Hunt
- Reddit 项目推荐

## 4. 触发与使用方式

手动触发。触发短语示例：
- "帮我挖选题"
- "最近有什么痛点"
- "找点小众项目"
- "搜一下 XX 领域的方案"

## 5. 产出物

输出到当前工作目录或用户指定目录：`gem-hunt-YYYY-MM-DD.md`

报告结构见 `templates/hunt-report.md`，核心包含：
- 痛点卡片列表
- 项目卡片列表
- 交叉匹配：哪些项目对应哪些痛点
- 每项附带"为什么值得关注"

Skill 止于报告。后续耳朵自行体验项目、决定是否推荐。

## 6. 文件结构

```
~/.claude/skills/gem-hunter/
├── SKILL.md                       # 主文件：persona + 触发 + 工作流
├── references/
│   ├── pain-point-sources.md      # 痛点发现：信息源 + 搜索策略 + 判断标准
│   ├── project-sources.md         # 方案发现：信息源 + 搜索条件 + 判断标准
│   └── matching-logic.md          # 交叉匹配逻辑
└── templates/
    └── hunt-report.md             # 输出模板
```

## 7. SKILL.md 设计要点

### 7.1 Persona 风格

中文，对话式。参考 video-spec-builder 的"废才"风格——有性格、直白、不说废话。但定位不同：废才是导演，猎手是侦察兵。语气冷静、务实、不迎合。

### 7.2 核心技能

- **信息源扫描**：能同时搜多个源，交叉验证信号
- **痛点识别**：区分"一次性抱怨"和"结构性痛点"
- **项目评估**：不看 Star 数，看解决问题的能力
- **交叉匹配**：把痛点和对策连起来
- **结构化输出**：按模板出报告

### 7.3 工作流

1. 识别用户意图（痛点 / 项目 / 都要）
2. 加载对应 references
3. 执行搜索扫描
4. 过滤 + 评估
5. 交叉匹配（如果两路都跑了）
6. 按模板输出报告

## 8. v1 范围

- 纯 prompt 驱动，不写 Python 脚本
- 依赖 Claude Code 的 WebSearch + WebFetch 能力
- 人机协作：skill 做扫描和初筛，耳朵做最终判断
- v2 考虑加 Python 脚本做系统化源聚合

## 9. 对接

本 skill 输出是终点。不自动调 video-spec-builder 或 jianying-editor。耳朵体验项目后，如需出视频，手动触发对应 skill。
