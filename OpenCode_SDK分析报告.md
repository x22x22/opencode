# OpenCode SDK 分析报告

## 摘要

本报告详细分析了 OpenCode 项目是否提供面向第三方系统的 SDK，以及其对流式对话和多 Agent 调度的支持能力。

**核心发现：**
- ✅ OpenCode 提供了官方 JavaScript/TypeScript SDK（`@opencode-ai/sdk`）
- ✅ 完整支持流式对话（基于 Server-Sent Events）
- ✅ 完整支持多 Agent 调度和管理
- ⚠️ 目前仅提供 JavaScript/TypeScript SDK，暂无 Python、Go、Java 等其他语言的官方 SDK
- ✅ 提供 VSCode 扩展作为集成示例

---

## 一、SDK 概述

### 1.1 JavaScript/TypeScript SDK

**包名称：** `@opencode-ai/sdk`  
**当前版本：** 1.1.61  
**许可证：** MIT  
**NPM 地址：** https://www.npmjs.com/package/@opencode-ai/sdk  
**位置：** `/packages/sdk/js/`

#### 安装方式

```bash
npm install @opencode-ai/sdk
# 或者
bun install @opencode-ai/sdk
# 或者
pnpm install @opencode-ai/sdk
# 或者
yarn add @opencode-ai/sdk
```

#### SDK 架构

OpenCode SDK 采用客户端-服务器架构，提供两个版本：

1. **v1 API** (`@opencode-ai/sdk`)
   - 完整的客户端和服务器功能
   - 成熟稳定的 API

2. **v2 API** (`@opencode-ai/sdk/v2`)
   - 改进的流式响应处理
   - 更现代的 API 设计
   - 完全向后兼容

#### 核心模块

```typescript
// v1 API
import { 
  createOpencode,           // 创建完整的 OpenCode 实例（服务器+客户端）
  createOpencodeClient,     // 仅创建客户端
  createOpencodeServer,     // 仅创建服务器
  createOpencodeTui         // 创建终端 UI
} from '@opencode-ai/sdk'

// v2 API
import { 
  createOpencode,
  createOpencodeClient,
  createOpencodeServer
} from '@opencode-ai/sdk/v2'
```

---

## 二、流式对话能力分析

### 2.1 流式响应机制

OpenCode SDK **完全支持流式对话**，基于标准的 **Server-Sent Events (SSE)** 协议实现。

#### 流式端点

1. **全局事件流**
   ```
   GET /global/event
   Content-Type: text/event-stream
   ```

2. **会话事件流**
   ```
   GET /event/subscribe
   Content-Type: text/event-stream
   ```

3. **会话消息流式响应**
   ```
   POST /session/{sessionID}/message
   支持流式返回 AI 响应
   ```

### 2.2 流式对话实现方式

#### 方法一：使用 v2 API 的 prompt 方法（推荐）

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk/v2'

const client = createOpencodeClient({ 
  baseUrl: 'http://localhost:4096' 
})

// 创建会话
const session = await client.session.create({
  directory: '/path/to/project'
})

// 发送消息并流式接收响应
const response = await client.session.prompt({
  sessionID: session.data.id,
  agent: 'build',  // 指定使用的 agent
  parts: [
    {
      type: 'text',
      text: '请帮我分析这个函数的性能问题'
    },
    {
      type: 'file',
      mime: 'text/plain',
      url: 'file:///path/to/file.js'
    }
  ]
})

// response 包含流式返回的完整消息
console.log(response.data.info)  // 消息元数据
console.log(response.data.parts) // 消息内容部分
```

#### 方法二：订阅事件流

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk/v2'

const client = createOpencodeClient({ 
  baseUrl: 'http://localhost:4096' 
})

// 订阅会话事件流
const eventStream = await client.event.subscribe({
  sessionID: session.id
})

// 处理流式事件
for await (const event of eventStream.stream) {
  switch (event.type) {
    case 'message.created':
      console.log('新消息:', event.data)
      break
    case 'message.updated':
      console.log('消息更新:', event.data)
      break
    case 'session.status':
      console.log('会话状态:', event.data)
      break
    // ... 更多事件类型
  }
}
```

### 2.3 支持的消息部分类型

OpenCode SDK 支持多种消息部分类型，实现丰富的对话场景：

```typescript
// 1. 文本部分
type TextPartInput = {
  id?: string
  type: 'text'
  text: string
  source?: {
    value: string
    start: number
    end: number
  }
}

// 2. 文件部分
type FilePartInput = {
  id?: string
  type: 'file'
  mime: string
  filename?: string
  url: string
  source?: FilePartSource
}

// 3. Agent 部分（用于多 Agent 调度）
type AgentPartInput = {
  id?: string
  type: 'agent'
  name: string
  source?: {
    value: string
    start: number
    end: number
  }
}

// 4. 子任务部分（用于任务分解）
type SubtaskPartInput = {
  id?: string
  type: 'subtask'
  prompt: string
  description: string
  agent: string
  model?: {
    providerID: string
    modelID: string
  }
  command?: string
}
```

### 2.4 流式对话特性

- ✅ **实时响应**：基于 SSE 协议，实时推送 AI 生成内容
- ✅ **多模态输入**：支持文本、文件、代码片段等多种输入
- ✅ **上下文保持**：会话级别的上下文管理
- ✅ **中断控制**：支持随时中止会话 (`session.abort()`)
- ✅ **消息历史**：完整的消息历史记录和回溯
- ✅ **Fork 机制**：可从任意消息点 fork 新会话

---

## 三、多 Agent 调度能力分析

### 3.1 Agent 架构

OpenCode 提供了**完整的多 Agent 调度系统**，包括：

1. **主 Agent（Primary Agents）**
   - `build`: 全权限开发 Agent（默认）
   - `plan`: 只读分析 Agent

2. **子 Agent（Subagents）**
   - `general`: 通用子 Agent，用于复杂搜索和多步骤任务
   - 自定义子 Agent 支持

3. **Agent 模式**
   - `primary`: 主 Agent 模式
   - `subagent`: 子 Agent 模式
   - `all`: 所有 Agent 模式

### 3.2 获取可用 Agent 列表

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk/v2'

const client = createOpencodeClient({ 
  baseUrl: 'http://localhost:4096' 
})

// 获取所有可用的 Agent
const agents = await client.app.agents({
  directory: '/path/to/project'
})

// agents 返回结构示例：
// [
//   {
//     id: 'build',
//     name: 'Build Agent',
//     description: '全权限开发 Agent',
//     mode: 'primary',
//     color: '#FF5733'
//   },
//   {
//     id: 'plan',
//     name: 'Plan Agent',
//     description: '只读分析 Agent',
//     mode: 'primary',
//     color: '#33C4FF'
//   },
//   {
//     id: 'general',
//     name: 'General Subagent',
//     description: '通用子 Agent',
//     mode: 'subagent',
//     color: '#44FF33'
//   }
// ]
```

### 3.3 指定 Agent 执行任务

#### 方式一：在会话创建时指定 Agent

```typescript
// 创建使用特定 Agent 的会话
const session = await client.session.create({
  directory: '/path/to/project',
  agent: 'plan'  // 使用 plan Agent
})
```

#### 方式二：在发送消息时指定 Agent

```typescript
// 使用不同的 Agent 处理不同的消息
const response1 = await client.session.prompt({
  sessionID: session.id,
  agent: 'build',  // 使用 build Agent 进行开发
  parts: [
    { type: 'text', text: '实现一个新功能' }
  ]
})

const response2 = await client.session.prompt({
  sessionID: session.id,
  agent: 'plan',   // 切换到 plan Agent 进行分析
  parts: [
    { type: 'text', text: '分析代码结构' }
  ]
})
```

### 3.4 多 Agent 协作：使用 Agent Part

OpenCode 支持在消息中直接调用其他 Agent，实现 Agent 之间的协作：

```typescript
// 在消息中引用另一个 Agent
const response = await client.session.prompt({
  sessionID: session.id,
  agent: 'build',
  parts: [
    {
      type: 'text',
      text: '我需要先分析这段代码，然后进行优化'
    },
    {
      type: 'agent',
      name: 'plan',  // 调用 plan Agent 进行分析
      source: {
        value: '@plan 分析 src/utils.ts 的性能问题',
        start: 0,
        end: 30
      }
    }
  ]
})
```

### 3.5 子任务分解与调度

使用 `SubtaskPartInput` 实现复杂任务的自动分解和多 Agent 调度：

```typescript
const response = await client.session.prompt({
  sessionID: session.id,
  agent: 'build',
  parts: [
    {
      type: 'text',
      text: '重构整个模块'
    },
    {
      type: 'subtask',
      prompt: '分析现有代码结构',
      description: '使用 plan Agent 分析代码',
      agent: 'plan',
      model: {
        providerID: 'anthropic',
        modelID: 'claude-3-5-sonnet-20241022'
      }
    },
    {
      type: 'subtask',
      prompt: '实现新的架构',
      description: '使用 build Agent 重构代码',
      agent: 'build',
      model: {
        providerID: 'anthropic',
        modelID: 'claude-3-5-sonnet-20241022'
      }
    }
  ]
})
```

### 3.6 Agent 切换机制

OpenCode 提供了灵活的 Agent 切换能力：

```typescript
// 通过 TUI 命令切换 Agent
await client.tui.execute({
  command: {
    type: 'agent_cycle'  // 切换到下一个 Agent
  }
})

await client.tui.execute({
  command: {
    type: 'agent_cycle_reverse'  // 切换到上一个 Agent
  }
})
```

### 3.7 Agent 配置与权限

```typescript
// 更新会话的 Agent 配置
await client.session.update({
  path: { id: session.id },
  body: {
    agent: 'build',
    permissions: {
      // 配置 Agent 的工具权限
      'bash': true,
      'edit': true,
      'create': true,
      'view': true
    }
  }
})
```

---

## 四、实际应用示例

### 4.1 完整的流式对话示例

```typescript
import { createOpencode } from '@opencode-ai/sdk/v2'

// 创建 OpenCode 实例（自动启动服务器）
const { client, server } = await createOpencode({
  hostname: '127.0.0.1',
  port: 4096,
  config: {
    logLevel: 'info'
  }
})

try {
  // 1. 创建会话
  const session = await client.session.create({
    directory: '/path/to/project'
  })
  console.log('会话已创建:', session.data.id)

  // 2. 订阅事件流
  const eventStream = client.event.subscribe({
    sessionID: session.data.id
  })

  // 3. 在后台处理事件流
  ;(async () => {
    for await (const event of eventStream.stream) {
      if (event.type === 'message.updated') {
        console.log('AI 回复中...', event.data.content)
      }
    }
  })()

  // 4. 发送消息
  const response = await client.session.prompt({
    sessionID: session.data.id,
    agent: 'build',
    parts: [
      {
        type: 'text',
        text: '请帮我创建一个 REST API 端点'
      }
    ]
  })

  console.log('任务完成:', response.data)

} finally {
  // 清理资源
  server.close()
}
```

### 4.2 多 Agent 协作示例

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk/v2'

const client = createOpencodeClient({ 
  baseUrl: 'http://localhost:4096' 
})

// 场景：代码审查和重构
async function codeReviewAndRefactor() {
  // 创建会话
  const session = await client.session.create({
    directory: '/path/to/project'
  })

  // 第一步：使用 plan Agent 进行代码审查
  console.log('步骤 1: 代码审查...')
  const reviewResponse = await client.session.prompt({
    sessionID: session.data.id,
    agent: 'plan',
    parts: [
      {
        type: 'text',
        text: '请审查 src/ 目录下的所有代码，找出性能问题和安全隐患'
      }
    ]
  })

  console.log('审查结果:', reviewResponse.data)

  // 第二步：使用 build Agent 根据审查结果进行重构
  console.log('步骤 2: 代码重构...')
  const refactorResponse = await client.session.prompt({
    sessionID: session.data.id,
    agent: 'build',
    parts: [
      {
        type: 'text',
        text: '根据上述审查结果，修复所有发现的问题'
      }
    ]
  })

  console.log('重构完成:', refactorResponse.data)

  // 第三步：再次使用 plan Agent 验证修改
  console.log('步骤 3: 验证修改...')
  const verifyResponse = await client.session.prompt({
    sessionID: session.data.id,
    agent: 'plan',
    parts: [
      {
        type: 'text',
        text: '验证刚才的修改是否正确解决了问题'
      }
    ]
  })

  console.log('验证结果:', verifyResponse.data)

  return session
}

codeReviewAndRefactor()
```

### 4.3 批量文件处理示例

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk/v2'
import { pathToFileURL } from 'node:url'
import { glob } from 'glob'

const client = createOpencodeClient({ 
  baseUrl: 'http://localhost:4096' 
})

// 批量为所有 TypeScript 文件生成测试
async function generateTests() {
  const files = await glob('src/**/*.ts')
  
  // 并发处理多个文件
  const tasks = files.map(async (file) => {
    // 为每个文件创建独立的会话
    const session = await client.session.create({
      directory: process.cwd()
    })

    console.log(`处理 ${file}...`)
    
    await client.session.prompt({
      sessionID: session.data.id,
      agent: 'build',
      parts: [
        {
          type: 'file',
          mime: 'text/plain',
          url: pathToFileURL(file).href
        },
        {
          type: 'text',
          text: '为这个文件中的所有导出函数编写单元测试'
        }
      ]
    })

    console.log(`完成 ${file}`)
    return file
  })

  await Promise.all(tasks)
  console.log('所有测试生成完成！')
}

generateTests()
```

---

## 五、其他语言的 SDK 支持情况

### 5.1 Python SDK

**状态：** ❌ **目前不存在**

OpenCode 目前**没有官方 Python SDK**。不过，由于 OpenCode 提供了基于 REST API 的接口（OpenAPI 3.1.1 规范），第三方开发者可以：

1. **使用 OpenAPI 生成器**
   ```bash
   openapi-generator generate \
     -i https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json \
     -g python \
     -o ./opencode-python-sdk
   ```

2. **手动封装 REST API**
   ```python
   import requests
   import sseclient
   
   class OpencodeClient:
       def __init__(self, base_url="http://localhost:4096"):
           self.base_url = base_url
       
       def create_session(self, directory):
           response = requests.post(
               f"{self.base_url}/session",
               json={"directory": directory}
           )
           return response.json()
       
       def send_message(self, session_id, text, agent="build"):
           response = requests.post(
               f"{self.base_url}/session/{session_id}/message",
               json={
                   "agent": agent,
                   "parts": [{"type": "text", "text": text}]
               }
           )
           return response.json()
       
       def subscribe_events(self, session_id):
           response = requests.get(
               f"{self.base_url}/event/subscribe",
               params={"sessionID": session_id},
               stream=True
           )
           client = sseclient.SSEClient(response)
           for event in client.events():
               yield event
   ```

### 5.2 Go SDK

**状态：** ❌ **目前不存在**

同样可以使用 OpenAPI 生成器或手动封装：

```bash
openapi-generator generate \
  -i https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json \
  -g go \
  -o ./opencode-go-sdk
```

### 5.3 Java SDK

**状态：** ❌ **目前不存在**

```bash
openapi-generator generate \
  -i https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json \
  -g java \
  -o ./opencode-java-sdk
```

### 5.4 Rust SDK

**状态：** ❌ **目前不存在**

```bash
openapi-generator generate \
  -i https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json \
  -g rust \
  -o ./opencode-rust-sdk
```

### 5.5 VSCode 扩展

**状态：** ✅ **已提供**

位置: `/sdks/vscode/`

OpenCode 提供了官方的 VSCode 扩展，展示了如何在 IDE 中集成 OpenCode：

- 快捷键 `Cmd+Esc` (Mac) 或 `Ctrl+Esc` (Windows/Linux) 打开终端
- 快捷键 `Cmd+Shift+Esc` (Mac) 或 `Ctrl+Shift+Esc` 启动新会话
- 自动共享当前选中的代码或文件
- 文件引用快捷方式 `Cmd+Option+K` (Mac) 或 `Alt+Ctrl+K`

---

## 六、API 完整性分析

### 6.1 核心 API 端点

OpenCode SDK 基于 OpenAPI 3.1.1 规范，提供以下核心能力：

#### 全局管理
- ✅ `GET /global/health` - 健康检查
- ✅ `GET /global/event` - 全局事件流（SSE）
- ✅ `GET /global/config` - 获取全局配置
- ✅ `PATCH /global/config` - 更新全局配置
- ✅ `POST /global/dispose` - 释放资源

#### 会话管理
- ✅ `GET /session` - 列出所有会话
- ✅ `POST /session` - 创建新会话
- ✅ `GET /session/{id}` - 获取会话详情
- ✅ `PATCH /session/{id}` - 更新会话属性
- ✅ `DELETE /session/{id}` - 删除会话
- ✅ `GET /session/{id}/children` - 获取子会话
- ✅ `POST /session/{id}/fork` - Fork 会话
- ✅ `POST /session/{id}/abort` - 中止会话
- ✅ `GET /session/{id}/diff` - 获取会话的代码差异
- ✅ `POST /session/{id}/summarize` - 总结会话

#### 消息管理
- ✅ `GET /session/{id}/message` - 列出会话消息
- ✅ `POST /session/{id}/message` - 发送消息（流式响应）
- ✅ `GET /session/{id}/message/{messageID}` - 获取特定消息
- ✅ `POST /session/{id}/prompt_async` - 异步发送消息

#### Agent 管理
- ✅ `GET /agent` - 列出所有可用 Agent

#### 文件操作
- ✅ `GET /file` - 列出文件
- ✅ `GET /file/read` - 读取文件内容
- ✅ `GET /file/status` - 获取文件状态

#### 搜索功能
- ✅ `POST /find/text` - 文本搜索
- ✅ `POST /find/files` - 文件搜索
- ✅ `POST /find/symbols` - 符号搜索

#### 权限管理
- ✅ `GET /session/{id}/permissions` - 列出权限请求
- ✅ `POST /session/{id}/permissions/{permissionID}` - 响应权限请求

#### PTY（伪终端）管理
- ✅ `GET /pty` - 列出 PTY 会话
- ✅ `POST /pty` - 创建 PTY 会话
- ✅ `DELETE /pty/{id}` - 删除 PTY 会话
- ✅ `GET /pty/{id}` - 获取 PTY 信息
- ✅ `PATCH /pty/{id}` - 更新 PTY
- ✅ `GET /pty/{id}/connect` - 连接到 PTY

#### MCP（Model Context Protocol）集成
- ✅ `POST /mcp` - 添加 MCP 服务器
- ✅ `GET /mcp/status` - 获取 MCP 状态
- ✅ `POST /mcp/{serverID}/connect` - 连接 MCP 服务器
- ✅ `POST /mcp/{serverID}/disconnect` - 断开 MCP 连接

#### LSP（Language Server Protocol）支持
- ✅ `GET /lsp/status` - 获取 LSP 状态

### 6.2 流式响应端点

以下端点支持 `text/event-stream` 格式的流式响应：

1. ✅ `GET /global/event` - 全局事件
2. ✅ `GET /event/subscribe` - 会话事件
3. ✅ `POST /session/{sessionID}/message` - 消息响应（通过 SSE）

---

## 七、技术架构特点

### 7.1 客户端-服务器架构

```
┌─────────────────────────────────────────────────────┐
│                   第三方应用                         │
│  (Web App / CLI / Mobile / Desktop / IDE Extension) │
└────────────────┬────────────────────────────────────┘
                 │ HTTP/SSE
                 ▼
┌─────────────────────────────────────────────────────┐
│              @opencode-ai/sdk                       │
│              (JavaScript/TypeScript)                │
└────────────────┬────────────────────────────────────┘
                 │ REST API
                 ▼
┌─────────────────────────────────────────────────────┐
│              OpenCode Server                         │
│              (localhost:4096)                        │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│            Agent Runtime                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│  │  build  │  │  plan   │  │ general │  ...       │
│  └─────────┘  └─────────┘  └─────────┘            │
└─────────────────────────────────────────────────────┘
```

### 7.2 核心优势

1. **Provider 无关性**
   - 支持 Anthropic Claude
   - 支持 OpenAI
   - 支持 Google Gemini
   - 支持本地模型

2. **开放架构**
   - 100% 开源
   - 标准 REST API
   - OpenAPI 规范
   - 易于集成

3. **强大的工具集成**
   - LSP（Language Server Protocol）支持
   - MCP（Model Context Protocol）集成
   - Git 集成
   - 文件系统操作
   - 终端模拟器

4. **灵活的部署方式**
   - 本地服务器
   - 远程服务器
   - Desktop 应用
   - TUI (Terminal UI)
   - VSCode 扩展

---

## 八、使用场景建议

### 8.1 适合的场景

✅ **代码生成与重构**
- 使用 `build` Agent 进行代码实现
- 使用 `plan` Agent 进行架构分析
- 支持批量文件处理

✅ **代码审查与分析**
- 使用 `plan` Agent 进行只读分析
- 安全的代码探索
- 性能分析和优化建议

✅ **自动化开发工作流**
- 批量测试生成
- 文档自动生成
- CI/CD 集成

✅ **IDE 集成**
- VSCode 扩展示例
- 自定义 IDE 插件
- 编辑器集成

✅ **多步骤任务自动化**
- 使用子任务分解
- 多 Agent 协作
- 复杂工作流编排

### 8.2 当前限制

⚠️ **仅支持 JavaScript/TypeScript**
- 其他语言需要自行封装或使用 OpenAPI 生成器

⚠️ **需要本地安装 OpenCode CLI**
- `createOpencodeServer()` 依赖本地 `opencode` 命令
- 或者连接到已有的 OpenCode 服务器

⚠️ **流式响应处理**
- 需要理解 SSE 协议
- 需要处理事件流的生命周期管理

---

## 九、总结与建议

### 9.1 核心结论

1. ✅ **OpenCode 完全支持流式对话**
   - 基于 Server-Sent Events 的实时流式响应
   - 支持多模态输入（文本、文件、代码）
   - 完整的会话管理和消息历史

2. ✅ **OpenCode 完全支持多 Agent 调度**
   - 内置主 Agent（build, plan）和子 Agent（general）
   - 支持动态 Agent 切换
   - 支持 Agent 间协作（AgentPartInput）
   - 支持复杂任务的子任务分解（SubtaskPartInput）

3. ⚠️ **SDK 语言支持有限**
   - 仅官方支持 JavaScript/TypeScript
   - 其他语言需要基于 OpenAPI 规范自行实现

### 9.2 对第三方系统的建议

#### 如果您使用 JavaScript/TypeScript

**推荐方案：**
```bash
npm install @opencode-ai/sdk
```

直接使用官方 SDK，享受完整的类型支持和流式响应处理。

#### 如果您使用其他编程语言

**方案一：使用 OpenAPI Generator（推荐）**
```bash
# 自动生成客户端代码
openapi-generator generate \
  -i https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json \
  -g <语言> \
  -o ./opencode-sdk
```

**方案二：手动封装 REST API**
- 参考 OpenAPI 规范实现 HTTP 客户端
- 实现 SSE 事件流处理
- 参考官方 JavaScript SDK 的实现逻辑

#### 集成步骤

1. **安装 OpenCode CLI**
   ```bash
   curl -fsSL https://opencode.ai/install | bash
   ```

2. **启动 OpenCode 服务器**
   ```bash
   opencode serve --hostname=127.0.0.1 --port=4096
   ```

3. **连接并使用**
   - 使用 SDK 连接到 `http://localhost:4096`
   - 创建会话
   - 发送消息
   - 处理流式响应

### 9.3 未来展望

根据项目架构和 OpenAPI 规范的完整性，OpenCode 有潜力支持更多编程语言的 SDK。建议社区开发者可以：

1. 基于 OpenAPI 规范开发其他语言的 SDK
2. 为 Python、Go、Java、Rust 等主流语言提供社区版 SDK
3. 向官方仓库贡献 SDK 实现

---

## 附录

### A. 相关资源

- **官方网站**: https://opencode.ai
- **GitHub 仓库**: https://github.com/anomalyco/opencode
- **NPM 包**: https://www.npmjs.com/package/@opencode-ai/sdk
- **文档**: https://opencode.ai/docs
- **Discord 社区**: https://opencode.ai/discord

### B. OpenAPI 规范位置

- **文件路径**: `/packages/sdk/openapi.json`
- **版本**: OpenAPI 3.1.1
- **在线访问**: https://github.com/anomalyco/opencode/raw/dev/packages/sdk/openapi.json

### C. 示例代码位置

- **JavaScript SDK 示例**: `/packages/sdk/js/example/example.ts`
- **VSCode 扩展**: `/sdks/vscode/`

---

**报告生成时间**: 2026-02-12  
**OpenCode 版本**: 1.1.61  
**分析范围**: `/packages/sdk/` 和相关目录
