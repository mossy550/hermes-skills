# Agent Hub MCP - OpenClaw 技能

## 功能描述

**Agent Hub MCP** 是通用多智能体协调平台，通过 Model Context Protocol (MCP) 实现任意 MCP 兼容 AI 智能体之间的协作通信。

### 核心能力
- **多智能体通信**：支持跨项目、跨平台的多智能体消息传递
- **特性协作**：基于 Feature 的多智能体项目协调（Feature → Task → Delegation → Subtask）
- **任务委托**：智能体之间可互相委托工作，跟踪进度
- **持久化记忆**：所有协作历史跨会话持久化存储
- **自动发现**：可自动检测项目能力并注册

### 技术特点
- **纯本地框架**：不依赖外部 LLM API，数据存储在本地
- **MCP 标准化**：符合 Model Context Protocol 规范
- **零配置启动**：支持 `npx -y agent-hub-mcp@latest` 一行命令启动
- **多存储后端**：支持文件存储（默认）和索引存储

---

## MCP 协议集成方式

### 启动方式

Agent Hub MCP 作为 MCP 服务器运行，通过 stdio 传输协议与 OpenClaw 集成。

```bash
# 方式1：npx 直接运行（推荐）
npx -y agent-hub-mcp@latest

# 方式2：本地安装后运行
npm install -g agent-hub-mcp
agent-hub-mcp

# 方式3：指定数据目录
AGENT_HUB_DATA_DIR=~/.agent-hub npx -y agent-hub-mcp@latest
```

### OpenClaw 配置

在 OpenClaw 的 MCP 配置中添加 agent-hub 服务器：

```json
{
  "mcpServers": {
    "agent-hub": {
      "command": "npx",
      "args": ["-y", "agent-hub-mcp@latest"],
      "env": {
        "AGENT_HUB_DATA_DIR": "/home/moss/.agent-hub"
      }
    }
  }
}
```

---

## 工具列表（Tools）

### 智能体管理

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `register_agent` | 注册/重连智能体 | `id`, `capabilities`, `collaborationPreferences` |
| `get_hub_status` | 获取 Hub 活动概览 | 无 |
| `sync` | 综合同步：消息+工作负载+Hub状态 | `agentId`（可选） |

### 消息通信

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `send_message` | 向其他智能体发送消息 | `to`, `content`, `type`, `priority`, `metadata` |
| `get_messages` | 获取消息，支持过滤 | `agentId`, `unreadOnly`, `type`, `since` |

### 特性协作（Feature Collaboration）

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `create_feature` | 创建多智能体协作项目 | `name`, `title`, `description`, `priority`, `estimatedAgents` |
| `create_task` | 将特性拆分为委托任务 | `featureId`, `title`, `delegations` |
| `create_subtask` | 创建实现步骤子任务 | `featureId`, `taskId`, `title`, `delegationId` |
| `accept_delegation` | 接受被委托的工作 | `delegationId` |
| `update_subtask` | 更新子任务状态和输出 | `featureId`, `subtaskId`, `status`, `output` |
| `get_features` | 列出特性，支持过滤 | `status`, `agentId`, `priority` |
| `get_feature` | 获取完整特性数据 | `featureId` |
| `get_agent_workload` | 获取智能体所有工作分配 | `agentId` |

---

## 资源列表（Resources）

Agent Hub MCP 提供以下 MCP 资源：

```
agent-hub://messages/{agentId}      # 智能体消息队列
agent-hub://agents/{agentId}        # 智能体注册信息
agent-hub://features/{featureId}     # 特性协作数据
agent-hub://status                  # Hub 状态
```

---

## 消息类型

| 类型 | 用途 |
|------|------|
| `context` | 共享状态/配置 |
| `task` | 分配工作 |
| `question` | 请求信息 |
| `completion` | 任务完成通知 |
| `error` | 错误报告 |
| `sync` | 实时同步请求 |

---

## 配置方法

### 环境变量

| 变量 | 默认值 | 描述 |
|------|--------|------|
| `AGENT_HUB_DATA_DIR` | `~/.agent-hub` | 数据存储目录 |

### 数据目录结构

```
~/.agent-hub/
├── messages/              # 智能体消息队列
│   └── {agent-id}.json
├── agents/                # 智能体注册数据
│   └── {agent-id}.json
└── features/              # 特性协作数据
    └── {feature-id}/
        ├── feature.json
        ├── tasks.json
        ├── delegations.json
        └── subtasks.json
```

### HTTP 调试模式（开发用）

```bash
# 启动 HTTP 服务器（默认端口 3737）
DEBUG=agent-hub:* npx -y agent-hub-mcp@latest
# 然后访问 http://localhost:3737 查看调试面板
```

---

## 使用示例

### 注册智能体

```
register_agent({
  id: "my-project-agent",
  capabilities: ["code-generation", "testing"],
  collaborationPreferences: { notifyOnMessage: true }
})
```

### 发送消息给其他智能体

```
send_message({
  to: "backend-agent",
  content: "需要 API 设计文档",
  type: "question",
  priority: "normal"
})
```

### 创建多智能体协作特性

```
create_feature({
  name: "user-auth-system",
  title: "用户认证系统",
  description: "实现登录、注册和会话管理",
  priority: "high",
  estimatedAgents: ["frontend-agent", "backend-agent"]
})
```

### 获取工作负载

```
sync()
```

---

## 系统要求

- **Node.js**: 22+
- **MCP 兼容客户端**: Claude Code, Qwen, Gemini CLI, Codex 等
- **操作系统**: Windows, macOS, Linux

---

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| MCP 服务器无法连接 | 重启 AI 助手 |
| 命令无法识别 | 检查自定义命令安装目录 |
| 智能体 ID 冲突 | 每个项目使用唯一 ID |
| 数据未持久化 | 检查 `AGENT_HUB_DATA_DIR` 目录权限 |

---

## 参考资料

- GitHub: https://github.com/gilbarbara/agent-hub-mcp
- System Overview: `./docs/SYSTEM-OVERVIEW.md`
- Troubleshooting: `./docs/TROUBLESHOOTING.md`
