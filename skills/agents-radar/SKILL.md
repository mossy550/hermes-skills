# agents-radar Skill

> AI 生态情报聚合 — 每日从 10 个数据源自动抓取、LLM 摘要，输出双语（中文 + 英文）情报简报。

## 功能概述

本技能将 [duanyytop/agents-radar](https://github.com/duanyytop/agents-radar) 框架的能力封装为 OpenClaw 可调用的本地工具。

核心能力：

- **多源情报聚合**：覆盖 GitHub / ArXiv / HN / HuggingFace / Product Hunt / Dev.to / Lobste.rs / Claude Code Skills / Anthropic&OpenAI 官网 / GitHub Trending 共 10 个数据源
- **纯本地运行**：无需 LLM API Key（使用托管 MCP 服务）；本地运行也支持 Anthropic / OpenAI / GitHub Copilot / OpenRouter
- **双语输出**：每份报告自动生成中英文版本
- **定时 + 按需**：GitHub Actions 每日 08:00 CST 运行；也支持本地手动触发
- **多维度报告**：CLI 工具对比、Agent 生态深度报告、Web 内容追踪、Trending 榜单、Hacker News 社区舆情、ArXiv 论文、HuggingFace 模型榜、Product Hunt 新品等

---

## 接入方式

### 方式 A：通过托管 MCP 服务（推荐，无需 API Key）

已在 OpenClaw 中配置好 HTTP MCP Server：

```
https://agents-radar-mcp.duanyytop.workers.dev
```

工具列表：

| 工具 | 说明 | 参数 |
|------|------|------|
| `list_reports` | 列出最近 N 天所有报告的日期和类型 | `days?: number`（默认 7） |
| `get_latest` | 获取最新一份指定类型报告 | `type: ReportType` |
| `get_report` | 获取指定日期 + 类型的报告 | `date: string, type: ReportType` |
| `search` | 关键词搜索近期报告 | `query: string, days?: number` |

`ReportType` 可选值：`cli` | `agents` | `web` | `trending` | `hn` | `ph` | `arxiv` | `hf` | `community` | `weekly` | `monthly`

#### 使用示例

```
# 获取最新 CLI 工具情报
"What's the latest in AI CLI tools?"

# 搜索 Claude Code 相关内容
"Search for Claude Code mentions this week"

# 获取指定日期的 Trending 报告
"Show me the AI trending report for 2026-03-05"
```

---

### 方式 B：本地安装运行（需 LLM API Key）

```bash
# 1. 克隆仓库
git clone https://github.com/duanyytop/agents-radar.git
cd agents-radar

# 2. 安装依赖
pnpm install

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 填入 LLM API Key

# 4. 运行
pnpm start
```

#### 本地环境变量

```bash
# LLM Provider（可选，默认 anthropic）
LLM_PROVIDER=anthropic   # anthropic | openai | github-copilot | openrouter

# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxxxx
ANTHROPIC_BASE_URL=https://api.kimi.com/coding/   # 用于 Kimi Code

# OpenAI
OPENAI_API_KEY=sk-xxxxx
OPENAI_BASE_URL=https://api.openai.com/           # 可选

# OpenRouter
OPENROUTER_API_KEY=sk-or-xxxxx

# GitHub Token（GitHub Actions 自动提供；本地需手动填入）
GITHUB_TOKEN=ghp_xxxxx

# 通知（可选）
TELEGRAM_BOT_TOKEN=xxxxx
TELEGRAM_CHAT_ID=xxxxx
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxxxx
```

---

## 数据源详解

| 数据源 | 类型 | 内容 | 报告类型 |
|--------|------|------|----------|
| **GitHub Repos** | API | 17+ AI 工具的 Issues、PRs、Releases | `cli` |
| **Claude Code Skills** | GitHub API | 社区热力排序的 Skills（按讨论度） | `cli` |
| **GitHub Trending** | HTML + API | 每日 trending + 7天内 AI 主题搜索（llm/ai-agent/rag/vector-db/ml） | `trending` |
| **Hacker News** | Algolia API | 过去24h Top 30 AI 相关故事 | `hn` |
| **Product Hunt** | GraphQL API | 昨日 AI 产品（按投票排序） | `ph` |
| **ArXiv** | ArXiv API | cs.AI / cs.CL / cs.LG 最近48h 论文 | `arxiv` |
| **HuggingFace** | Hub API | 过去7天热力模型（按 likes 排序） | `hf` |
| **Dev.to** | Forem API | 5个 AI/LLM 标签下热门文章 | `community` |
| **Lobste.rs** | JSON API | 过去7天 AI/ML 标签故事 | `community` |
| **Anthropic 官网** | Sitemap | /news/ /research/ /engineering/ /learn/ 新文章 | `web` |
| **OpenAI 官网** | Sitemap | research/publication/release/company 等新文章 | `web` |

---

## 输出格式

报告文件写入 `digests/YYYY-MM-DD/`，每种报告生成中英文各一份：

| 文件 | 内容 | Issue 标签 |
|------|------|------------|
| `ai-cli.md` / `ai-cli-en.md` | CLI 工具横向对比 + 单工具详情 + Skills 热力榜 | `digest` |
| `ai-agents.md` / `ai-agents-en.md` | OpenClaw 深度报告 + 10 个同类项目横向对比 | `openclaw` |
| `ai-web.md` / `ai-web-en.md` | Anthropic + OpenAI 官网新文章（仅在有新内容时生成） | `web` |
| `ai-trending.md` / `ai-trending-en.md` | GitHub AI Trending 分类榜单 + 趋势信号 | `trending` |
| `ai-hn.md` / `ai-hn-en.md` | Hacker News Top 30 AI 故事 + 舆情分析 | `hn` |
| `ai-ph.md` / `ai-ph-en.md` | Product Hunt 昨日 AI 新品（需 PRODUCTHUNT_TOKEN） | `ph` |
| `ai-arxiv.md` / `ai-arxiv-en.md` | ArXiv 关键论文摘要 | `arxiv` |
| `ai-hf.md` / `ai-hf-en.md` | HuggingFace 热力模型榜 | `hf` |
| `ai-community.md` / `ai-community-en.md` | Dev.to + Lobste.rs 社区文章汇总 | `community` |
| `ai-weekly.md` / `ai-weekly-en.md` | 周报（每周一生成，覆盖上周） | `weekly` |
| `ai-monthly.md` / `ai-monthly-en.md` | 月报（每月1日生成） | `monthly` |

---

## 定时调度

| 工作流 | Cron（UTC） | 北京时间 |
|--------|------------|----------|
| 每日简报 | `0 0 * * *` | 每日 08:00 |
| 周报 | `0 1 * * 1` | 每周一 09:00 |
| 月报 | `0 2 1 * *` | 每月1日 10:00 |

---

## 自定义配置

编辑 `config.yml` 即可增删跟踪的仓库，无需改代码：

```yaml
# 新增 CLI 工具
cli_repos:
  - id: my-tool
    repo: owner/my-ai-cli
    name: My AI Tool

# 新增同类项目
openclaw_peers:
  - id: my-agent
    repo: owner/my-agent
    name: My Agent
```

---

## 技术栈

- **Runtime**: Node.js + TypeScript（ESM）
- **LLM 集成**: Anthropic SDK / OpenAI SDK，支持 Provider 抽象
- **部署**: GitHub Actions（定时）+ Cloudflare Workers（MCP 服务）
- **通知**: Telegram Bot / 飞书 Webhook / 小红书 / 微信公众号

---

## 注意事项

1. MCP 服务（方式 A）完全免费，无需任何 API Key，数据每日更新
2. 本地运行（方式 B）需要 LLM API Key，默认使用 Anthropic claude-sonnet-4-20250514
3. Web 内容为增量检测，首次运行会抓取较多历史文章（约25篇/源）
4. Product Hunt 报告需要 `PRODUCTHUNT_TOKEN` 环境变量
