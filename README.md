# 🏔️ Gem Hunter（猎手）

小众开源项目选题发现器 —— Claude Code skill。

## 核心能力

- **两条独立流水线**：痛点发现（X/Twitter + Reddit + HN + GitHub + 小红书 + V2EX）和方案发现（Fossick MCP + 社区验证）
- **交叉匹配**：把痛点和对策连起来，标注匹配度，输出"痛点→方案"配对报告
- **非开发者过滤器**：四道闸（粒度、语言、门槛、场景）过滤掉开发者工具噪音

## 目录结构

```
SKILL.md              # Skill 主文件（Claude Code 加载入口）
references/
  pain-point-sources.md   # 痛点发现：信息源与搜索策略（API 调用格式 + 判断标准）
  project-sources.md      # 方案发现：信息源与搜索策略（Fossick MCP + 社区验证）
  matching-logic.md       # 交叉匹配逻辑
templates/
  hunt-report.md          # 选题报告输出模板
docs/
  specs/                  # 设计文档
```

## 数据采集原则

**不走搜索引擎中转论坛内容。直接调平台 API，拿到带互动数据的一手讨论。**

| 优先级 | 源 | 方式 |
|--------|-----|------|
| 🔴 最高 | X/Twitter | Bird 引擎（GraphQL API 直连） |
| 🟠 高 | Reddit | JSON API（需代理；被封锁时降级 WebSearch） |
| 🟡 中高 | HN | Algolia API |
| 🟢 中 | GitHub | Fossick MCP / gh CLI |
| ⚠️ 兜底 | 小红书 | WebSearch（MCP API 待修复） |
| ⚠️ 兜底 | V2EX | WebSearch（无公开 API） |

## 安装

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/erduo1998-cell/gem-hunter-skill.git gem-hunter
```

依赖：
- Node.js（X/Twitter Bird 引擎）
- curl + python3（Reddit/HN API 解析）
- AUTH_TOKEN/CT0（X/Twitter 认证，从 last30days .env 读取）
- ClashX 代理（`127.0.0.1:7890`）

## 使用

在 Claude Code 中直接说：
- "帮我挖选题" → 两条流水线都跑
- "最近有什么痛点" → 只跑痛点发现
- "找个能解决 session 管理的小众项目" → 只跑方案发现
- "搜一下 AI agent 操作电脑有没有新方案" → 关键词限定

输出报告保存到：`gem-hunt-YYYY-MM-DD.md`
