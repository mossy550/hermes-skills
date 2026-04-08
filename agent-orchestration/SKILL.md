---
name: agent-orchestration
description: MCP多智能体协作框架，支持共享内存池、任务队列、资源锁、研究优先工作流。与OpenClaw subagent机制深度整合，实现多智能体自动调度。
tags: [mcp, multi-agent, orchestration, coordination, shared-memory, task-queue]
permissions: [network, shell]
metadata:
  clawdbot:
    requires:
      bins: [node, npm, npx]
      env: []
    files: []
  capabilities:
    allow:
      - execute: [node, npm, npx]
      - network: [npmjs.com, registry.npmjs.org]
      - read: [workspace/**]
      - write: [workspace/**]
    deny:
      - execute: ["!node", "!npm", "!npx"]
      - network: ["!npmjs.com", "!registry.npmjs.org"]
  env_declarations:
    - name: MCP_ORCH_DB_PATH
      required: false
      default: ".agent-orchestration/orchestrator.db"
      description: SQLite数据库路径（每项目独立）
    - name: MCP_ORCH_SYNC_CONTEXT
      required: false
      default: "false"
      description: 自动同步activeContext.md上下文文件
    - name: MCP_ORCH_AGENT_NAME
      required: false
      default: "auto-generated"
      description: 默认智能体名称
    - name: MCP_ORCH_AGENT_ROLE
      required: false
      default: "sub"
      description: 默认智能体角色（main/sub）
    - name: MCP_ORCH_CAPABILITIES
      required: false
      default: "code"
      description: 智能体能力标签，逗号分隔

---

# Agent Orchestration — MCP多智能体协作技能

## 概述

`agent-orchestration` 是一个基于 **Model Context Protocol (MCP)** 的多智能体协作框架，使多个AI智能体能够共享内存、协调任务、有效协作。

**核心特性：** 纯本地框架，不依赖外部LLM API；每项目独立SQLite存储；支持任意支持MCP或AGENTS.md的IDE/CLI智能体（Cursor、VS Code Copilot、Aider、Windsurf等）。

---

## 核心机制

### 1. 共享内存池（Shared Memory）

智能体通过命名空间键值存储共享上下文，持久化到每项目SQLite数据库。

**命名空间结构：**

| 命名空间 | 用途 | 示例键 |
|----------|------|--------|
| `context` | 当前状态和焦点 | `current_focus`, `current_branch` |
| `decisions` | 架构决策 | `auth_strategy`, `db_choice` |
| `findings` | 分析结果 | `perf_issues`, `security_audit` |
| `blockers` | 阻塞问题 | `api_down`, `missing_deps` |
| `research:<task_id>:context` | 任务代码库理解 | - |
| `research:<task_id>:files` | 影响的文件 | - |
| `research:<task_id>:requirements` | 规格和边界情况 | - |
| `research:<task_id>:design` | 架构决策 | - |
| `delegation:<task_id>:brief` | Cursor委托简报 | - |
| `delegation:<task_id>:updates` | 委托更新 | - |
| `delegation:<task_id>:findings` | 委托发现 | - |
| `delegation:<task_id>:decisions` | 委托决策 | - |
| `delegation:<task_id>:handoff` | 委托交接 | - |

### 2. 任务队列（Task Queue）

基于复杂度的任务管理，支持研究优先工作流：

| 复杂度 | 示例 | 研究要求 |
|--------|------|----------|
| `trivial` | 拼写修复、配置变更 | 无 |
| `simple` | Bug修复、小重构 | context + files |
| `moderate` | 新端点、组件 | + requirements |
| `complex` | 新功能、迁移 | + design |

**任务状态机：** `pending` → `in_progress` → `completed` / `blocked`

### 3. 资源锁（Resource Locking）

防止并发访问冲突，确保文件编辑排他性。

### 4. 智能体注册与发现

智能体注册、心跳保活、自动清理离线智能体（5分钟无心跳）。

### 5. 研究优先工作流（Research-First）

非平凡任务必须完成研究阶段才能开始实现，确保智能体理解上下文后再编码。

---

## MCP工具集

### 会话管理

| 工具 | 说明 |
|------|------|
| `bootstrap` | 初始化会话：注册、获取焦点、任务、决策 |
| `claim_todo` | 子智能体：一步完成注册+创建/认领任务 |
| `agent_whoami` | 获取当前智能体信息 |

### 智能体管理

| 工具 | 说明 |
|------|------|
| `agent_register` | 注册智能体 |
| `agent_heartbeat` | 发送心跳保活 |
| `agent_list` | 列出所有已注册智能体 |
| `agent_unregister` | 注销智能体（释放所有锁） |

### 共享内存

| 工具 | 说明 |
|------|------|
| `memory_set` | 存储键值到共享内存 |
| `memory_get` | 从共享内存检索值 |
| `memory_list` | 列出命名空间内所有键 |
| `memory_delete` | 删除共享内存值 |

### 任务管理

| 工具 | 说明 |
|------|------|
| `task_create` | 创建新任务（自动检测复杂度） |
| `task_claim` | 认领任务（研究未完成则阻塞） |
| `task_update` | 更新任务状态或进度 |
| `task_complete` | 标记任务完成 |
| `task_list` | 按过滤器列出任务 |
| `is_my_turn` | 检查是否有可认领工作 |

### 研究工作流

| 工具 | 说明 |
|------|------|
| `research_ready` | 标记任务研究完成 |
| `research_status` | 检查任务研究状态 |
| `research_query` | 搜索历史研究发现 |
| `research_checklist` | 按复杂度查看研究要求 |

### 协调

| 工具 | 说明 |
|------|------|
| `lock_acquire` | 获取资源锁 |
| `lock_release` | 释放持有的锁 |
| `lock_check` | 检查资源是否被锁 |
| `coordination_status` | 获取整体系统状态 |

### Cursor委托（可选）

| 工具 | 说明 |
|------|------|
| `cursor_check` | 验证Cursor CLI可用性 |
| `cursor_delegate_task` | 启动Cursor CLI运行任务并持久化会话元数据 |
| `cursor_task_status` | 检查委托任务健康度和恢复状态 |
| `cursor_resume_task` | 返回委托任务的恢复命令 |
| `cursor_sync_task` | 刷新委托任务状态并将发现同步回共享内存 |
| `cursor_recover_task` | 重新启动失败/过期的委托任务 |
| `cursor_handoff_task` | 为下一个智能体记录结构化交接 |
| `cursor_list_delegations` | 显示委托的Cursor任务及最后已知状态 |

---

## 协作流程

### 主智能体（Main Agent）

```
1. bootstrap                              # 启动会话
2. memory_set current_focus "..."         # 设置项目焦点
3. task_create "Feature X"                # 创建任务（复杂度自动检测）
4. task_create "Feature Y"
5. coordination_status                    # 监控进度
```

### 子智能体（Sub-Agent）

```
1. claim_todo "Feature X"                # 注册 + 查看研究清单

# 研究阶段（非平凡任务）
2. memory_set key="understanding" namespace="research:<task_id>:context" value="..."
3. memory_set key="files" namespace="research:<task_id>:files" value="..."
4. research_ready task_id="<task_id>"     # 验证研究完成

# 实现阶段
5. task_claim task_id="<task_id>"        # 现在允许开始
6. lock_acquire "src/feature.ts"         # 编辑前先锁文件
7. [do the work]
8. task_complete <task_id> "Done"       # 完成
9. agent_unregister                       # 清理
```

### 简单任务（无需研究）

```
1. claim_todo "Fix typo in README"       # 复杂度：trivial
2. task_claim task_id="<task_id>"         # 立即允许
3. [do the work]
4. task_complete <task_id> "Done"
```

---

## OpenClaw集成配置

### 方式一：npx直接调用（推荐）

在项目目录下初始化：

```bash
cd /path/to/project
npx agent-orchestration init          # 创建AGENTS.md（兼容任意AI智能体）
npx agent-orchestration serve         # 运行MCP服务器
```

### 方式二：MCP配置集成

在OpenClaw的MCP配置（`~/.openclaw/mcp.json`）中添加：

```json
{
  "mcpServers": {
    "agent-orchestration": {
      "command": "npx",
      "args": ["-y", "agent-orchestration", "serve"],
      "env": {
        "MCP_ORCH_SYNC_CONTEXT": "true",
        "MCP_ORCH_DB_PATH": ".agent-orchestration/orchestrator.db"
      }
    }
  }
}
```

### 方式三：项目级Cursor配置

在项目`.cursor/mcp.json`中添加：

```json
{
  "mcpServers": {
    "agent-orchestration": {
      "command": "npx",
      "args": ["-y", "agent-orchestration", "serve"],
      "env": {
        "MCP_ORCH_SYNC_CONTEXT": "true"
      }
    }
  }
}
```

---

## 项目级配置（可选）

在项目根目录创建 `agent-orchestration.config.json`：

```json
{
  "cursor": {
    "binary": "agent",
    "defaultMode": "agent",
    "defaultForce": true,
    "autoApproveMcps": true,
    "trustWorkspace": true,
    "useCreateChat": true,
    "logDir": ".agent-orchestration/providers/cursor",
    "preferWorktreeFor": ["moderate", "complex"],
    "recoveryStaleAfterMs": 600000
  }
}
```

---

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MCP_ORCH_DB_PATH` | `.agent-orchestration/orchestrator.db` | SQLite数据库路径 |
| `MCP_ORCH_SYNC_CONTEXT` | `false` | 自动同步activeContext.md |
| `MCP_ORCH_AGENT_NAME` | 自动生成 | 默认智能体名称 |
| `MCP_ORCH_AGENT_ROLE` | `sub` | 默认智能体角色 |
| `MCP_ORCH_CAPABILITIES` | `code` | 智能体能力标签 |

---

## 依赖安装

```bash
# 全局安装
npm install -g agent-orchestration

# 或直接使用npx（无需安装）
npx -y agent-orchestration serve

# 本地项目安装
cd /path/to/project
npm install agent-orchestration
```

**前置要求：** Node.js 18+

---

## 数据库与存储

- **存储位置：** `.agent-orchestration/orchestrator.db`（每项目独立）
- **数据库损坏处理：** `rm -rf .agent-orchestration/` （下次启动自动重建）
- **自动清理：** 5分钟无心跳的离线智能体自动注销

---

## 自动文档生成

任务文档自动生成到 `.agent-orchestration/docs/`：

- `.agent-orchestration/docs/tasks/<task-id>.md`
- `.agent-orchestration/docs/README.md`

触发时机：委托启动/同步/交接/恢复/任务完成

手动生成：
```bash
npx agent-orchestration doc --task <task-id>
```

---

## 与OpenClaw subagent机制的结合

本技能可作为OpenClaw多智能体协作的底层协调层：

1. **主智能体**（中枢）负责任务分解、调度
2. **子智能体**（各部门）通过`agent-orchestration`共享内存协调
3. **共享内存**作为跨智能体通信的骨干
4. **任务队列**确保按优先级顺序执行

```
┌──────────────────────────────────────┐
│  OpenClaw Main Agent（中枢/尚书省）    │
│  - 任务分解与调度                     │
│  - 调用 agent_orchestration MCP工具   │
└──────────────┬───────────────────────┘
               │ MCP (共享内存 + 任务队列)
┌──────────────▼───────────────────────┐
│  agent-orchestration MCP Server       │
│  - SQLite per-project DB             │
│  - 多智能体协调中枢                   │
└──────────────┬───────────────────────┘
               │
    ┌──────────┼──────────┬──────────┐
    ▼          ▼          ▼          ▼
 SubAgent1  SubAgent2  SubAgent3  SubAgentN
```

---

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| 服务器无法启动 | 确认Node.js 18+：`node --version` |
| 数据库错误 | 删除`.agent-orchestration/`目录，重建 |
| 智能体互相看不见 | 确认所有智能体使用相同`cwd`；检查`agent_list` |
| 锁冲突 | 使用`lock_check`确认资源状态；超时自动释放 |
