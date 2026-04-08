# open-multi-agent-framework

TypeScript 多智能体编排框架，**一次 `runTeam()` 调用即可从目标到结果** —— 框架自动将目标分解为任务 DAG、解决依赖关系，并行运行智能体。

3 个运行时依赖 · 33 个源文件 · 部署在任意 Node.js 运行环境 · 支持 Anthropic / OpenAI / Grok / Gemini / 本地模型（Ollama / vLLM / LM Studio）

> **关键特性：** 纯本地框架，不依赖外部 LLM API（通过 Ollama 等本地模型实现）。支持任务自动分解、并行执行、结构化输出、任务重试、人工审批、循环检测、可观测性追踪。

---

## 安装依赖

### 方式一：npm 安装（推荐）

```bash
npm install @jackchen_me/open-multi-agent
```

### 方式二：从源码构建

```bash
git clone https://github.com/JackChen-me/open-multi-agent.git
cd open-multi-agent
npm install
npm run build
```

### 运行时依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `@anthropic-ai/sdk` | ^0.52.0 | Anthropic (Claude) 适配器 |
| `openai` | ^4.73.0 | OpenAI (GPT) + OpenAI 兼容 API 适配器 |
| `zod` | ^3.23.0 | 结构化输出 Schema 验证 |

### 可选依赖

| 包 | 安装命令 | 用途 |
|----|----------|------|
| `@google/genai` | `npm install @google/genai` | Gemini 适配器（可选） |

### 环境变量

```bash
# 云端模型（按需设置）
export ANTHROPIC_API_KEY=sk-...
export OPENAI_API_KEY=sk-...
export GEMINI_API_KEY=...
export XAI_API_KEY=...       # Grok
export GITHUB_TOKEN=...      # GitHub Copilot

# 本地模型（无需 API key）
# Ollama 默认 http://localhost:11434/v1
# 其他本地服务器（vLLM / LM Studio / llama.cpp）同理
```

---

## 核心 API

### 入口类：`OpenMultiAgent`

```typescript
import { OpenMultiAgent } from '@jackchen_me/open-multi-agent'

const orchestrator = new OpenMultiAgent({
  defaultModel: 'claude-sonnet-4-6',   // 默认模型
  maxConcurrency: 5,                   // 最大并发数（默认5）
  onProgress: (event) => console.log(event.type, event.agent ?? event.task),
})
```

### 三种运行模式

| 模式 | 方法 | 适用场景 |
|------|------|----------|
| 单智能体 | `runAgent()` | 单一智能体、单一提示词 |
| 自动编排团队 | `runTeam()` | 给定目标，框架自动规划和执行 |
| 显式任务管道 | `runTasks()` | 你定义任务图和分配 |

---

## API 详解

### 1. `runTeam()` — 自动编排团队

框架自动将目标分解为任务 DAG，分配给合适的智能体，并行执行，汇总最终结果。

```typescript
import { OpenMultiAgent } from '@jackchen_me/open-multi-agent'
import type { AgentConfig } from '@jackchen_me/open-multi-agent'

const orchestrator = new OpenMultiAgent({
  defaultModel: 'claude-sonnet-4-6',
  onProgress: (event) => console.log(event.type, event.agent ?? event.task),
})

const team = orchestrator.createTeam('api-team', {
  name: 'api-team',
  agents: [
    {
      name: 'architect',
      model: 'claude-sonnet-4-6',
      systemPrompt: '你是软件架构师，设计清晰的 API 契约和文件结构。',
      tools: ['bash', 'file_write'],
    },
    {
      name: 'developer',
      model: 'claude-sonnet-4-6',
      systemPrompt: '你是 TypeScript 开发者，实现架构师设计的方案。',
      tools: ['bash', 'file_read', 'file_write', 'file_edit'],
    },
    {
      name: 'reviewer',
      model: 'claude-sonnet-4-6',
      systemPrompt: '你是高级代码审查员，审查正确性、安全性和可读性。',
      tools: ['bash', 'file_read', 'grep'],
    },
  ],
  sharedMemory: true,  // 智能体间共享内存
})

const result = await orchestrator.runTeam(team, '在 /tmp/todo-api/ 创建一个 TODO 列表 REST API')

console.log(`Success: ${result.success}`)
console.log(`Output tokens: ${result.totalTokenUsage.output_tokens}`)
console.log(result.agentResults.get('coordinator')?.output)
```

**执行流程示例：**
```
agent_start   coordinator
task_start    architect
task_complete architect
task_start    developer
task_start    developer        // 独立任务并行执行
task_complete developer
task_start    reviewer          // 依赖完成后解锁
task_complete developer
task_complete reviewer
agent_complete coordinator
```

---

### 2. `runTasks()` — 显式任务管道

你定义任务 DAG 和依赖关系，框架负责调度和执行。

```typescript
import { OpenMultiAgent, createTask } from '@jackchen_me/open-multi-agent'
import type { Task } from '@jackchen_me/open-multi-agent'

const orchestrator = new OpenMultiAgent({ defaultModel: 'claude-sonnet-4-6' })

const designTask = createTask({
  title: 'API Design',
  description: '设计 TODO API 的路由和数据模型',
  assignee: 'architect',
})

const implementTask = createTask({
  title: 'Implementation',
  description: '实现 TODO API',
  assignee: 'developer',
  dependsOn: [designTask.id],  // 依赖 designTask
})

const testTask = createTask({
  title: 'Test & Review',
  description: '测试并审查代码',
  assignee: 'reviewer',
  dependsOn: [implementTask.id],
})

const result = await orchestrator.runTasks(
  [designTask, implementTask, testTask],
  { model: 'claude-sonnet-4-6', tools: ['bash', 'file_read', 'file_write'] }
)
```

**任务重试配置：**
```typescript
const task = createTask({
  title: 'API Design',
  description: '...',
  assignee: 'architect',
  maxRetries: 3,        // 失败后重试3次
  retryDelayMs: 1000,   // 初始重试延迟 1 秒
  retryBackoff: 2,      // 指数退避乘数
})
```

---

### 3. `runAgent()` — 单智能体执行

```typescript
const orchestrator = new OpenMultiAgent()

const result = await orchestrator.runAgent(
  {
    name: 'assistant',
    model: 'claude-sonnet-4-6',
    systemPrompt: '你是一个有帮助的助手。',
    tools: ['bash', 'file_read'],
  },
  '用中文解释什么是 monad，要求100字以内。'
)

console.log(result.output)
```

---

### 4. 自定义工具

使用 `defineTool()` + Zod Schema 定义工具：

```typescript
import { z } from 'zod'
import { defineTool } from '@jackchen_me/open-multi-agent'

const fetchDataTool = defineTool({
  name: 'fetch_data',
  description: '从 URL 获取 JSON 数据',
  inputSchema: z.object({
    url: z.string().url(),
    params: z.record(z.string()).optional(),
  }),
  execute: async ({ url, params }) => {
    const res = await fetch(url + (params ? '?' + new URLSearchParams(params) : ''))
    const data = await res.json()
    return { data: JSON.stringify(data), isError: false }
  },
})
```

注册到工具注册表：
```typescript
import { Agent, ToolRegistry, ToolExecutor, registerBuiltInTools } from '@jackchen_me/open-multi-agent'

const registry = new ToolRegistry()
registerBuiltInTools(registry)      // 注册内置工具（bash/file_read/file_write 等）
registry.register(fetchDataTool)    // 注册自定义工具

const executor = new ToolExecutor(registry)
const agent = new Agent(agentConfig, registry, executor)
```

---

### 5. 循环检测

防止智能体陷入重复调用：

```typescript
const agentConfig: AgentConfig = {
  name: 'developer',
  model: 'claude-sonnet-4-6',
  systemPrompt: '...',
  tools: ['bash', 'file_read'],
  loopDetection: {
    maxRepetitions: 3,       // 同一调用重复3次触发检测
    loopDetectionWindow: 4,  // 追踪最近4轮
    onLoopDetected: 'warn',  // 'warn' | 'terminate' | (info) => 'continue'|'inject'|'terminate'
  },
}
```

---

### 6. 结构化输出

用 Zod Schema 约束输出格式：

```typescript
import { z } from 'zod'
import { OpenMultiAgent } from '@jackchen_me/open-multi-agent'

const schema = z.object({
  title: z.string(),
  summary: z.string(),
  steps: z.array(z.string()),
})

const orchestrator = new OpenMultiAgent({ defaultModel: 'claude-sonnet-4-6' })

const result = await orchestrator.runAgent(
  {
    name: 'planner',
    model: 'claude-sonnet-4-6',
    systemPrompt: '你是一个任务规划专家。',
    outputSchema: schema,  // 结果会被解析为 JSON
  },
  '创建一个学习 React 的计划'
)

// result.structured 是类型安全的 JS 对象
const plan = result.structured as z.infer<typeof schema>
console.log(plan.title, plan.steps)
```

---

### 7. 本地模型（Ollama / vLLM / LM Studio / llama.cpp）

通过 `provider: 'openai'` + `baseURL` 使用任意 OpenAI 兼容的本地模型：

```typescript
const localAgent: AgentConfig = {
  name: 'local-reviewer',
  model: 'llama3.1',
  provider: 'openai',
  baseURL: 'http://localhost:11434/v1',  // Ollama
  // baseURL: 'http://localhost:8000/v1',  // vLLM
  // baseURL: 'http://localhost:1234/v1',  // LM Studio
  // baseURL: 'http://localhost:8080/v1',  // llama.cpp
  apiKey: 'ollama',        // 占位符，Ollama 会忽略
  tools: ['file_read', 'grep'],
  timeoutMs: 120_000,      // 本地推理慢，设置超时
}
```

**本地 + 云端混合团队：**
```typescript
const team = orchestrator.createTeam('hybrid-team', {
  name: 'hybrid-team',
  agents: [
    { name: 'coder',   model: 'claude-sonnet-4-6', provider: 'anthropic', ... },
    { name: 'reviewer', model: 'llama3.1', provider: 'openai', baseURL: 'http://localhost:11434/v1', apiKey: 'ollama', ... },
  ],
  sharedMemory: true,
})
```

---

### 8. 内置工具

| 工具 | 说明 |
|------|------|
| `bash` | 执行 Shell 命令，返回 stdout + stderr |
| `file_read` | 读取文件内容，支持 offset/limit |
| `file_write` | 写入/创建文件，自动创建父目录 |
| `file_edit` | 精确字符串替换编辑文件 |
| `grep` | 正则搜索文件内容 |

**工具预设：**
```typescript
const readonlyAgent = { toolPreset: 'readonly' }   // file_read, grep, glob
const readwriteAgent = { toolPreset: 'readwrite' } // file_read, file_write, file_edit, grep, glob
const fullAgent = { toolPreset: 'full' }            // + bash
```

---

### 9. 可观测性（Trace）

```typescript
const orchestrator = new OpenMultiAgent({
  defaultModel: 'claude-sonnet-4-6',
  onTrace: (trace) => {
    console.log(`[${trace.type}] ${trace.runId}`, trace.data)
  },
})
```

---

### 10. 生命周期钩子

```typescript
const agentConfig: AgentConfig = {
  name: 'developer',
  model: 'claude-sonnet-4-6',
  beforeRun: async (context) => {
    console.log(`Running: ${context.prompt}`)
    return context  // 可修改 prompt 后返回
  },
  afterRun: async (result) => {
    console.log(`Done, output length: ${result.output.length}`)
    return result  // 可修改结果后返回
  },
}
```

---

## 完整示例：自动化任务团队

```typescript
import { OpenMultiAgent } from '@jackchen_me/open-multi-agent'
import type { AgentConfig, OrchestratorEvent } from '@jackchen_me/open-multi-agent'

const orchestrator = new OpenMultiAgent({
  defaultModel: 'claude-sonnet-4-6',
  maxConcurrency: 5,
  onProgress: (event: OrchestratorEvent) => {
    const ts = new Date().toISOString().slice(11, 23)
    switch (event.type) {
      case 'agent_start':    console.log(`[${ts}] AGENT START  → ${event.agent}`); break
      case 'agent_complete':  console.log(`[${ts}] AGENT DONE   ← ${event.agent}`); break
      case 'task_start':      console.log(`[${ts}] TASK START  ↓ ${event.task}`); break
      case 'task_complete':   console.log(`[${ts}] TASK DONE   ↑ ${event.task}`); break
      case 'error':           console.error(`[${ts}] ERROR       ✗ ${event.agent}`); break
    }
  },
})

const team = orchestrator.createTeam('research-team', {
  name: 'research-team',
  agents: [
    {
      name: 'researcher',
      model: 'claude-sonnet-4-6',
      provider: 'anthropic',
      systemPrompt: '你是一个研究专家，负责搜集和整理信息。',
      tools: ['file_write'],
    },
    {
      name: 'writer',
      model: 'claude-sonnet-4-6',
      provider: 'anthropic',
      systemPrompt: '你是一个技术作家，负责撰写清晰的技术文档。',
      tools: ['file_read', 'file_write'],
    },
  ],
  sharedMemory: true,
})

const result = await orchestrator.runTeam(
  team,
  '调研 TypeScript 5.6 的新特性，输出一份 markdown 文档到 /tmp/ts56-features.md'
)

if (result.success) {
  console.log('团队任务成功完成')
  console.log(`总 Token 消耗: input=${result.totalTokenUsage.input_tokens}, output=${result.totalTokenUsage.output_tokens}`)
} else {
  console.error('团队任务失败')
}
```

---

## OpenClaw 集成

将此框架注册为 OpenClaw 技能，在技能中直接 import 使用：

```typescript
// 技能文件示例：skills/open-multi-agent-framework/use.ts
import { OpenMultiAgent } from '@jackchen_me/open-multi-agent'
import type { AgentConfig } from '@jackchen_me/open-multi-agent'

export async function runResearchTeam(goal: string) {
  const orchestrator = new OpenMultiAgent({
    defaultModel: 'claude-sonnet-4-6',
  })

  const team = orchestrator.createTeam('research', {
    name: 'research',
    agents: [
      { name: 'researcher', model: 'claude-sonnet-4-6', systemPrompt: '...', tools: ['file_write'] },
      { name: 'writer', model: 'claude-sonnet-4-6', systemPrompt: '...', tools: ['file_read', 'file_write'] },
    ],
    sharedMemory: true,
  })

  return orchestrator.runTeam(team, goal)
}
```

---

## 支持的模型提供商

| 提供商 | 配置 | 环境变量 | 状态 |
|--------|------|----------|------|
| Anthropic (Claude) | `provider: 'anthropic'` | `ANTHROPIC_API_KEY` | ✅ 已验证 |
| OpenAI (GPT) | `provider: 'openai'` | `OPENAI_API_KEY` | ✅ 已验证 |
| Grok (xAI) | `provider: 'grok'` | `XAI_API_KEY` | ✅ 已验证 |
| GitHub Copilot | `provider: 'copilot'` | `GITHUB_TOKEN` | ✅ 已验证 |
| Gemini | `provider: 'gemini'` | `GEMINI_API_KEY` | ✅ 已验证 |
| Ollama / vLLM / LM Studio / llama.cpp | `provider: 'openai'` + `baseURL` | — | ✅ 已验证 |

> 任何 OpenAI 兼容 API（DeepSeek / Groq / Mistral / Qwen / MiniMax 等）均可通过 `provider: 'openai'` + `baseURL` 使用。

---

## 类型参考

主要类型导出：

```typescript
import type {
  AgentConfig,        // 智能体静态配置
  AgentRunResult,     // 单智能体运行结果
  TeamRunResult,      // 团队运行结果
  Task,               // 任务对象
  OrchestratorEvent,  // 进度事件
  ToolDefinition,      // 工具定义
  StreamEvent,        // 流式事件
  TokenUsage,         // Token 消耗
} from '@jackchen_me/open-multi-agent'
```

---

## 文件结构

```
@jackchen_me/open-multi-agent/
├── dist/                    # 编译输出
│   ├── index.js
│   └── index.d.ts
├── src/
│   ├── orchestrator/        # 顶层编排器（OpenMultiAgent 类）
│   ├── agent/               # 智能体层（Agent、AgentPool、LoopDetector）
│   ├── team/                # 团队层（Team、MessageBus）
│   ├── task/                # 任务层（TaskQueue、Task）
│   ├── tool/                # 工具系统（ToolRegistry、Built-in tools）
│   ├── llm/                 # LLM 适配器（Anthropic、OpenAI、Gemini...）
│   ├── memory/              # 内存（InMemoryStore、SharedMemory）
│   └── types.ts             # 所有公共类型定义
├── examples/                # 13 个可运行示例
│   ├── 01-single-agent.ts
│   ├── 02-team-collaboration.ts
│   ├── 03-task-pipeline.ts
│   ├── 04-multi-model-team.ts
│   ├── 05-copilot-test.ts
│   ├── 06-local-model.ts
│   ├── 07-fan-out-aggregate.ts
│   ├── 08-gemma4-local.ts
│   ├── 09-structured-output.ts
│   ├── 10-task-retry.ts
│   ├── 11-trace-observability.ts
│   ├── 12-grok.ts
│   └── 13-gemini.ts
└── package.json
```

运行示例：
```bash
npx tsx examples/02-team-collaboration.ts
```
