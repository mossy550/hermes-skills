# agents-radar OpenClaw Skill

## 快速使用

### 推荐：MCP 接入（零配置）

OpenClaw 已预配置托管 MCP 服务，**无需任何 API Key**：

```
MCP Server: https://agents-radar-mcp.duanyytop.workers.dev
```

直接询问即可获取情报：
- "最新 AI CLI 工具动态是什么？"
- "搜索过去一周 Claude Code 相关内容"
- "查看 2026-04-01 的 AI Trending 报告"

### 本地安装

```bash
git clone https://github.com/duanyytop/agents-radar.git
cd agents-radar
pnpm install
cp .env.example .env
# 填入 ANTHROPIC_API_KEY 或 OPENAI_API_KEY
pnpm start
```

## 数据源（10个）

| # | 数据源 | 报告类型 |
|---|--------|---------|
| 1 | GitHub Repos（17+ AI 工具） | CLI |
| 2 | Claude Code Skills | CLI |
| 3 | GitHub Trending | Trending |
| 4 | Hacker News | HN |
| 5 | Product Hunt | PH |
| 6 | ArXiv | ArXiv |
| 7 | HuggingFace Hub | HF |
| 8 | Dev.to | Community |
| 9 | Lobste.rs | Community |
| 10 | Anthropic + OpenAI 官网 | Web |

## 报告类型

`cli` · `agents` · `web` · `trending` · `hn` · `ph` · `arxiv` · `hf` · `community` · `weekly` · `monthly`
