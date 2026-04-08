# Hermes Skills

OpenClaw AI军团为Hermes Agent开发的技能库。

## 技能列表

| 技能 | 描述 | 依赖 |
|------|------|------|
| [agent-hub-mcp](./agent-hub-mcp) | 多智能体MCP协调服务器 | Node.js 22+ |
| [agent-orchestration](./agent-orchestration) | 多智能体协作框架，共享内存池 | Node.js 22+ |
| [agents-radar](./agents-radar) | AI生态情报聚合，10数据源 | MCP服务或本地 |
| [open-multi-agent-framework](./open-multi-agent-framework) | TypeScript多智能体框架 | npm |

## 安装方式

在Hermes中安装任意技能：
```bash
hermes skills install mossy550/hermes-skills/<技能名>
```

例如：
```bash
hermes skills install mossy550/hermes-skills/agent-hub-mcp
hermes skills install mossy550/hermes-skills/agent-orchestration
hermes skills install mossy550/hermes-skills/agents-radar
hermes skills install mossy550/hermes-skills/open-multi-agent-framework
```

## 详情

参见各技能目录下的 `SKILL.md`
