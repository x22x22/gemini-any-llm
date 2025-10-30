# Codex App-Server 功能教学级解读

## 目录

1. [概述](#概述)
2. [核心架构](#核心架构)
3. [Codex 提供商详解](#codex-提供商详解)
4. [认证机制](#认证机制)
5. [请求处理流程](#请求处理流程)
6. [流式响应机制](#流式响应机制)
7. [工具调用支持](#工具调用支持)
8. [推理能力](#推理能力)
9. [使用场景与示例](#使用场景与示例)
10. [配置指南](#配置指南)
11. [故障排查](#故障排查)

## 概述

### 什么是 Codex App-Server？

Codex App-Server 是 gemini-any-llm 项目中的一个核心功能模块，它是一个基于 NestJS 构建的应用服务器，作为 API 网关来连接 Gemini CLI 和 OpenAI 的 Codex 服务（gpt-5-codex 模型）。

**核心价值**：
- 🔌 **无缝集成**：让 Gemini CLI 能够访问 OpenAI Codex 模型，无需修改 CLI 代码
- 🔐 **多认证模式**：支持 API Key 和 ChatGPT OAuth 两种认证方式
- ⚡ **高性能流式**：完整支持 Server-Sent Events (SSE) 流式响应
- 🛠️ **工具调用**：完整的 Function Calling 支持
- 🧠 **推理能力**：支持 Codex 的推理（reasoning）功能，展示思考过程
- 🔄 **协议转换**：自动转换 Gemini API 格式与 Codex API 格式

### 技术栈

- **框架**：NestJS (基于 Node.js 和 TypeScript)
- **语言**：TypeScript 5.7+
- **运行时**：Node.js 18+
- **包管理**：pnpm
- **HTTP 客户端**：原生 fetch API
- **配置管理**：YAML + class-validator

## 核心架构

### 系统架构图

```
┌─────────────────┐
│   Gemini CLI    │
│  (用户界面)      │
└────────┬────────┘
         │ HTTP/HTTPS (Gemini API 格式)
         ↓
┌─────────────────────────────────────────────────────────┐
│                  Gemini Any LLM Gateway                  │
│                                                          │
│  ┌──────────────┐    ┌────────────────┐                │
│  │  Controllers │ →  │  Transformers  │                │
│  │  (HTTP 入口) │    │  (协议转换层)   │                │
│  └──────────────┘    └───────┬────────┘                │
│                              │                          │
│                              ↓                          │
│  ┌─────────────────────────────────────────┐           │
│  │         Codex Provider                  │           │
│  │  ┌──────────────────────────────────┐   │           │
│  │  │  - API 调用管理                  │   │           │
│  │  │  - 流式响应处理                  │   │           │
│  │  │  - 工具调用编排                  │   │           │
│  │  │  - 推理内容处理                  │   │           │
│  │  └──────────────────────────────────┘   │           │
│  │                                          │           │
│  │  ┌──────────────────────────────────┐   │           │
│  │  │  ChatGPT Auth Manager            │   │           │
│  │  │  - OAuth 2.0 流程                │   │           │
│  │  │  - Token 自动刷新                │   │           │
│  │  │  - 回调服务器管理                │   │           │
│  │  └──────────────────────────────────┘   │           │
│  └─────────────────────────────────────────┘           │
└─────────────────────┬───────────────────────────────────┘
                      │ HTTP/HTTPS (Codex API 格式)
                      ↓
            ┌──────────────────────┐
            │   OpenAI Codex API   │
            │ (gpt-5-codex model)  │
            └──────────────────────┘
```

### 模块组织

```
src/
├── controllers/           # HTTP 请求处理层
│   ├── gemini.controller.ts   # Gemini API 端点实现
│   └── health.controller.ts   # 健康检查端点
│
├── providers/            # AI 提供商实现层
│   ├── codex/
│   │   ├── codex.provider.ts        # Codex 核心逻辑
│   │   └── chatgpt-auth.manager.ts  # OAuth 认证管理
│   ├── openai/
│   └── claude-code/
│
├── transformers/         # 协议转换层
│   ├── request.transformer.ts      # 请求格式转换
│   ├── response.transformer.ts     # 响应格式转换
│   └── stream.transformer.ts       # 流式响应转换
│
├── models/              # 数据模型定义
│   ├── codex/
│   │   ├── codex-request.model.ts    # Codex 请求模型
│   │   └── codex-stream-event.model.ts # Codex 流事件模型
│   ├── openai/
│   └── gemini/
│
├── config/              # 配置管理
│   ├── config.module.ts
│   ├── config.schema.ts
│   └── global-config.service.ts
│
├── cli/                 # 命令行工具
│   ├── gal.ts              # 主入口
│   ├── gal-code.ts         # 对话命令
│   ├── gal-auth.ts         # 认证配置
│   └── gal-gateway.ts      # 网关管理
│
└── common/              # 共享资源
    └── prompts/
        └── gpt5-codex-instructions.md  # Codex 系统提示词
```

## Codex 提供商详解

### CodexProvider 类核心职责

`CodexProvider` 是 Codex 功能的核心实现类，位于 `src/providers/codex/codex.provider.ts`，主要职责包括：

1. **配置管理**：读取和验证 Codex 相关配置
2. **认证处理**：支持 API Key 和 OAuth 两种模式
3. **请求构建**：将 Gemini 格式转换为 Codex 格式
4. **流式响应**：处理 SSE 事件流并转换回 Gemini 格式
5. **工具调用**：管理 Function Calling 的完整生命周期
6. **推理内容**：处理和展示 Codex 的思考过程

### 关键特性

#### 1. 双认证模式

**API Key 模式**：
```typescript
// 配置示例
codex:
  authMode: ApiKey
  apiKey: "sk-xxx..."
  baseURL: "https://chatgpt.com/backend-api/codex"
  model: "gpt-5-codex"
```

**ChatGPT OAuth 模式**：
```typescript
// 配置示例
codex:
  authMode: ChatGPT
  baseURL: "https://chatgpt.com/backend-api/codex"
  model: "gpt-5-codex"
```

#### 2. 协议转换

Codex Provider 实现了完整的协议转换逻辑：

**Gemini → Codex 请求转换**：
```typescript
// Gemini 格式
{
  "messages": [
    { "role": "user", "content": "写一个 HTTP 服务" }
  ]
}

// 转换为 Codex 格式
{
  "model": "gpt-5-codex",
  "instructions": "You are Codex...",  // 系统提示词
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [
        { "type": "input_text", "text": "写一个 HTTP 服务" }
      ]
    }
  ],
  "stream": true,
  "store": false,
  "include": ["reasoning.encrypted_content"],
  "prompt_cache_key": "conversation-uuid"
}
```

**Codex → Gemini 响应转换**：
```typescript
// Codex SSE 事件
data: {"type":"response.output_text.delta","delta":"Hello"}

// 转换为 Gemini 流式响应
{
  "id": "resp-xxx",
  "object": "chat.completion.chunk",
  "created": 1234567890,
  "model": "gpt-5-codex",
  "choices": [{
    "index": 0,
    "delta": {
      "role": "assistant",
      "content": "Hello"
    }
  }]
}
```

#### 3. 推理内容处理

Codex 支持展示模型的思考过程（reasoning），Provider 会：

```typescript
// Codex 推理事件
data: {"type":"response.reasoning_text.delta","delta":"首先分析需求..."}

// 转换为 Gemini 格式
{
  "choices": [{
    "delta": {
      "reasoning_content": "首先分析需求..."
    }
  }]
}
```

配置推理参数：
```yaml
codex:
  reasoning:
    effort: medium    # 推理强度: low/medium/high
    summary: auto     # 推理摘要: auto/none
  textVerbosity: medium  # 输出冗余度: low/medium/high
```

#### 4. 工具调用编排

支持完整的 Function Calling 流程：

**工具定义转换**：
```typescript
// OpenAI 工具格式
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "获取天气信息",
    "parameters": { /* JSON Schema */ }
  }
}

// 转换为 Codex 格式
{
  "type": "function",
  "name": "get_weather",
  "description": "获取天气信息",
  "strict": false,
  "parameters": { /* 相同 */ }
}
```

**工具调用流程**：
1. 模型决定调用工具
2. Provider 捕获 `response.function_call_arguments.delta` 事件
3. 累积工具参数并转换为 Gemini 格式
4. 返回工具调用结果给模型
5. 模型基于结果继续生成

## 认证机制

### API Key 模式

最简单的认证方式，适合有 API Key 的场景：

```typescript
// 请求头
Authorization: Bearer sk-xxx...
originator: codex_cli_rs
version: 0.38.0
OpenAI-Beta: responses=experimental
```

### ChatGPT OAuth 模式

完整的 OAuth 2.0 PKCE 流程，由 `ChatGPTAuthManager` 管理。

#### OAuth 登录流程

```
1. 用户启动 gal code
   ↓
2. 检测到需要 OAuth 认证
   ↓
3. 生成 PKCE challenge
   ↓
4. 启动本地回调服务器 (127.0.0.1:1455)
   ↓
5. 显示授权 URL (用户在浏览器打开)
   https://auth.openai.com/oauth/authorize?
     client_id=app_EMoamEEZ73f0CkXaXp7hrann
     &redirect_uri=http://127.0.0.1:1455/callback
     &response_type=code
     &scope=openid%20profile%20email
     &code_challenge=xxx
     &code_challenge_method=S256
   ↓
6. 用户在浏览器完成授权
   ↓
7. 浏览器重定向到回调地址
   http://127.0.0.1:1455/callback?code=xxx
   ↓
8. 回调服务器接收 code
   ↓
9. 用 code 交换 tokens
   POST https://auth.openai.com/oauth/token
   {
     "grant_type": "authorization_code",
     "code": "xxx",
     "redirect_uri": "http://127.0.0.1:1455/callback",
     "code_verifier": "yyy",
     "client_id": "app_EMoamEEZ73f0CkXaXp7hrann"
   }
   ↓
10. 保存 tokens 到 ~/.gemini-any-llm/codex/auth.json
    {
      "tokens": {
        "id_token": "eyJ...",
        "access_token": "sess-...",
        "refresh_token": "ref-...",
        "account_id": "user-xxx"
      },
      "last_refresh": "2025-10-30T05:11:43.059Z",
      "access_token_expires_at": 1730270503059
    }
   ↓
11. 后续请求自动使用 access_token
    Authorization: Bearer sess-xxx...
    chatgpt-account-id: user-xxx
   ↓
12. Token 过期前自动刷新
```

#### Token 自动刷新

`ChatGPTAuthManager` 会在 Token 过期前 1 分钟自动刷新：

```typescript
private async refreshTokensInternal(tokens: ChatGPTTokens): Promise<ChatGPTTokens> {
  const response = await fetch(`${this.issuer}/oauth/token`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'User-Agent': this.buildUserAgent(),
    },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: tokens.refreshToken,
      client_id: this.clientId,
      codex_cli_simplified_flow: 'true',
      originator: 'codex_cli_rs',
    }),
  });
  // ... 处理响应并更新 tokens
}
```

#### 认证文件结构

```json
{
  "OPENAI_API_KEY": null,
  "tokens": {
    "id_token": "JWT token...",
    "access_token": "session token...",
    "refresh_token": "refresh token...",
    "account_id": "user-xxx"
  },
  "last_refresh": "2025-10-30T05:11:43.059Z",
  "access_token_expires_at": 1730270503059
}
```

## 请求处理流程

### 完整请求链路

```
用户输入
   ↓
┌──────────────────────────────────────────────┐
│ 1. Gemini CLI 发送请求                       │
│    POST /api/v1/models/gpt-5-codex:streamGenerateContent
│    Content-Type: application/json           │
│    {                                         │
│      "contents": [                           │
│        {                                     │
│          "role": "user",                     │
│          "parts": [{"text": "写个服务"}]     │
│        }                                     │
│      ]                                       │
│    }                                         │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 2. GeminiController 接收请求                │
│    - 提取 model 参数                         │
│    - 验证请求格式                            │
│    - 调用 RequestTransformer                │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 3. RequestTransformer 转换格式               │
│    Gemini → OpenAI 通用格式                  │
│    {                                         │
│      "messages": [                           │
│        {                                     │
│          "role": "user",                     │
│          "content": "写个服务"               │
│        }                                     │
│      ],                                      │
│      "model": "gpt-5-codex",                │
│      "stream": true                          │
│    }                                         │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 4. CodexProvider 处理请求                    │
│    a) buildCodexPayload 构建 Codex 请求     │
│       - 添加 instructions                   │
│       - 转换 messages → input               │
│       - 设置 reasoning 参数                 │
│    b) buildRequestHeaders 构建请求头        │
│       - API Key 或 OAuth Token              │
│       - 添加 conversation_id                │
│    c) streamCodexChunks 发送请求            │
│       - fetch Codex API                     │
│       - 处理重试逻辑                        │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 5. Codex API 返回 SSE 流                    │
│    data: {"type":"response.output_text.delta","delta":"def"}
│    data: {"type":"response.output_text.delta","delta":" serve"}
│    data: {"type":"response.completed","usage":{...}}
│    data: [DONE]                             │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 6. parseSSE 解析事件流                       │
│    - 按 \n\n 分割事件                        │
│    - 提取 data: 行                           │
│    - JSON 解析                               │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 7. mapEventToChunk 转换事件                  │
│    - 处理 delta 事件                         │
│    - 处理 reasoning 事件                     │
│    - 处理 function_call 事件                │
│    - 处理 completed 事件                     │
│    转换为 OpenAI 格式 chunk                  │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 8. StreamTransformer 转换流                  │
│    OpenAI chunk → Gemini SSE 格式            │
└──────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────┐
│ 9. 返回给 Gemini CLI                         │
│    data: [{"candidates":[{"content":{...}}]}]
└──────────────────────────────────────────────┘
   ↓
显示给用户
```

### 代码实现要点

#### 请求构建

```typescript
// src/providers/codex/codex.provider.ts
private buildCodexPayload(
  request: OpenAIRequest,
  config: ResolvedCodexConfig,
): CodexRequest {
  const { instructions, input } = this.transformMessages(request.messages);
  const tools = this.transformTools(request.tools);

  const payload: CodexRequest = {
    model: config.model,
    instructions,     // 系统提示词
    input,           // 转换后的消息列表
    stream: true,
    store: false,
    include: ['reasoning.encrypted_content'],
    prompt_cache_key: this.conversationId,
    parallel_tool_calls: false,
  };

  if (tools.length > 0) {
    payload.tools = tools;
  }

  if (config.reasoning) {
    payload.reasoning = { ...config.reasoning };
  }

  const verbosity = config.textVerbosity;
  if (verbosity) {
    payload.text = { verbosity };
  }

  return payload;
}
```

#### 消息转换

```typescript
private transformMessages(messages: OpenAIRequest['messages'] = []): {
  instructions: string;
  input: CodexInputItem[];
} {
  const instructions = GPT5_CODEX_BASE_INSTRUCTIONS;
  const input: CodexInputItem[] = [];

  for (const message of messages) {
    if (message.role === 'system' && message.content) {
      // System 消息包装为特殊格式
      input.push({
        type: 'message',
        role: 'user',
        content: [{
          type: 'input_text',
          text: `<system>${content}</system>`,
        }],
      });
      continue;
    }

    if (message.role === 'user') {
      input.push({
        type: 'message',
        role: 'user',
        content: [{
          type: 'input_text',
          text: message.content,
        }],
      });
      continue;
    }

    if (message.role === 'assistant') {
      // 处理工具调用
      if (Array.isArray(message.tool_calls)) {
        for (const toolCall of message.tool_calls) {
          input.push({
            type: 'function_call',
            call_id: toolCall.id,
            name: toolCall.function.name,
            arguments: toolCall.function.arguments,
          });
        }
      }

      // 处理普通内容
      if (message.content) {
        input.push({
          type: 'message',
          role: 'assistant',
          content: [{
            type: 'output_text',
            text: message.content,
          }],
        });
      }
      continue;
    }

    if (message.role === 'tool') {
      // 工具调用结果
      input.push({
        type: 'function_call_output',
        call_id: message.tool_call_id,
        output: message.content || '',
      });
    }
  }

  return { instructions, input };
}
```

## 流式响应机制

### SSE 事件类型

Codex API 返回多种 SSE 事件类型：

1. **文本内容事件**
   - `response.output_text.delta`: 增量文本
   - `response.output_item.done`: 文本项完成

2. **推理内容事件**
   - `response.reasoning_text.delta`: 推理过程增量
   - `response.reasoning_text.done`: 推理过程完成
   - `response.reasoning_summary_text.delta`: 推理摘要增量
   - `response.reasoning_summary_text.done`: 推理摘要完成

3. **工具调用事件**
   - `response.function_call_arguments.delta`: 工具参数增量
   - `response.output_item.done` (type=function_call): 工具调用完成

4. **完成事件**
   - `response.completed`: 整个响应完成

### 事件处理逻辑

```typescript
private mapEventToChunk(
  event: CodexStreamEvent,
  context: CodexStreamContext,
): OpenAIStreamChunk | null {
  const eventType = event.type ?? '';

  // 1. 处理推理事件
  if (eventType.includes('reasoning')) {
    return this.handleReasoningEvent(event, context);
  }

  // 2. 处理完成事件
  if (eventType === 'response.completed') {
    context.usage = this.extractUsage(event);
    context.completed = true;
    return this.createCompletionChunk(context);
  }

  // 3. 处理工具调用增量
  if (eventType === 'response.function_call_arguments.delta') {
    const { toolCall, state } = this.getOrCreateToolCall(
      context,
      event.delta.call_id,
      event.delta.name,
    );
    state.argumentsBuffer += event.delta.arguments;
    toolCall.function.arguments = state.argumentsBuffer;
    
    return {
      id: context.responseId,
      object: 'chat.completion.chunk',
      created: Math.floor(Date.now() / 1000),
      model: context.model,
      choices: [{
        index: 0,
        delta: {
          tool_calls: [{
            index: state.index,
            id: toolCall.id,
            type: 'function',
            function: {
              name: toolCall.function.name,
              arguments: event.delta.arguments,
            },
          }],
        },
      }],
    };
  }

  // 4. 处理文本增量
  const delta = this.extractDeltaText(event);
  if (delta) {
    context.accumulated += delta;
    return {
      id: context.responseId,
      object: 'chat.completion.chunk',
      created: Math.floor(Date.now() / 1000),
      model: context.model,
      choices: [{
        index: 0,
        delta: {
          role: context.started ? undefined : 'assistant',
          content: delta,
        },
      }],
    };
  }

  return null;
}
```

### 推理内容处理

```typescript
private handleReasoningEvent(
  event: CodexStreamEvent,
  context: CodexStreamContext,
): OpenAIStreamChunk | null {
  const text = this.extractReasoningText(event);
  if (!text) return null;

  // 避免重复发送
  const key = this.getReasoningStateKey(event);
  if (key) {
    const state = this.getOrCreateReasoningState(context, key);
    if (event.type.endsWith('.done') && state.hasStreamed) {
      return null;
    }
    state.hasStreamed = true;
  }

  return {
    id: context.responseId,
    object: 'chat.completion.chunk',
    created: Math.floor(Date.now() / 1000),
    model: context.model,
    choices: [{
      index: 0,
      delta: {
        role: context.started ? undefined : 'assistant',
        reasoning_content: text,  // 推理内容字段
      },
    }],
  };
}
```

## 工具调用支持

### 工具定义

支持标准的 OpenAI Function Calling 格式：

```typescript
const tools = [
  {
    type: 'function',
    function: {
      name: 'get_weather',
      description: '获取指定城市的天气信息',
      parameters: {
        type: 'object',
        properties: {
          city: {
            type: 'string',
            description: '城市名称，如"北京"',
          },
          unit: {
            type: 'string',
            enum: ['celsius', 'fahrenheit'],
            description: '温度单位',
          },
        },
        required: ['city'],
      },
    },
  },
];
```

### 工具调用流程

```
1. 用户: "北京今天天气怎么样？"
   ↓
2. 模型决定调用 get_weather 工具
   ↓
3. Codex 返回 function_call 事件
   data: {"type":"response.function_call_arguments.delta","delta":{"call_id":"call_123","name":"get_weather","arguments":"{\""}}
   data: {"type":"response.function_call_arguments.delta","delta":{"arguments":"city"}}
   data: {"type":"response.function_call_arguments.delta","delta":{"arguments":"\": \""}}
   data: {"type":"response.function_call_arguments.delta","delta":{"arguments":"北京"}}
   data: {"type":"response.function_call_arguments.delta","delta":{"arguments":"\"}"}}
   ↓
4. Provider 累积参数并返回完整工具调用
   {
     "choices": [{
       "delta": {
         "tool_calls": [{
           "id": "call_123",
           "type": "function",
           "function": {
             "name": "get_weather",
             "arguments": "{\"city\":\"北京\"}"
           }
         }]
       },
       "finish_reason": "tool_calls"
     }]
   }
   ↓
5. 客户端执行工具并返回结果
   {
     "role": "tool",
     "tool_call_id": "call_123",
     "content": "北京今天晴，温度 15-25°C"
   }
   ↓
6. 再次发送给模型，包含工具结果
   input: [
     { type: 'message', role: 'user', content: [...] },
     { type: 'function_call', call_id: 'call_123', name: 'get_weather', arguments: '{"city":"北京"}' },
     { type: 'function_call_output', call_id: 'call_123', output: '北京今天晴，温度 15-25°C' }
   ]
   ↓
7. 模型基于工具结果生成最终回复
   "根据天气信息，北京今天天气晴朗，温度在 15-25°C 之间，适合户外活动。"
```

### 工具状态管理

```typescript
interface ToolCallState {
  index: number;           // 工具调用索引
  argumentsBuffer: string; // 累积的参数字符串
  hasStreamed: boolean;    // 是否已流式输出
}

private getOrCreateToolCall(
  context: CodexStreamContext,
  callId: string,
  name?: string,
): { toolCall: OpenAIToolCall; state: ToolCallState } {
  let state = context.toolCallState.get(callId);
  
  if (!state) {
    // 创建新的工具调用
    const index = context.toolCalls.length;
    const toolCall: OpenAIToolCall = {
      id: callId,
      type: 'function',
      function: {
        name: name || 'function',
        arguments: '',
      },
    };
    context.toolCalls.push(toolCall);
    
    state = {
      index,
      argumentsBuffer: '',
      hasStreamed: false,
    };
    context.toolCallState.set(callId, state);
    
    return { toolCall, state };
  }

  // 返回已存在的工具调用
  const toolCall = context.toolCalls[state.index];
  if (name && !toolCall.function.name) {
    toolCall.function.name = name;
  }
  
  return { toolCall, state };
}
```

## 推理能力

### 推理模式配置

Codex 支持配置推理行为：

```yaml
codex:
  reasoning:
    effort: medium      # 推理强度
      # - low: 快速推理，适合简单任务
      # - medium: 平衡模式，大多数场景
      # - high: 深度推理，复杂问题
    summary: auto       # 推理摘要
      # - auto: 自动生成摘要
      # - none: 不生成摘要
  textVerbosity: medium # 输出详细度
    # - low: 简洁输出
    # - medium: 标准输出
    # - high: 详细输出
```

### 推理内容展示

推理内容通过特殊字段 `reasoning_content` 返回：

```typescript
// Gemini CLI 会识别并特殊展示这个字段
{
  "choices": [{
    "delta": {
      "reasoning_content": "让我分析一下这个问题..."
    }
  }]
}
```

### 推理事件去重

避免重复发送推理内容：

```typescript
interface CodexReasoningState {
  hasStreamed: boolean;
}

private getReasoningStateKey(event: CodexStreamEvent): string | null {
  const itemId = event.item_id;
  if (!itemId) return null;

  const outputIndex = event.output_index ?? 0;
  const contentIndex = event.content_index ?? 0;

  return `${itemId}:${outputIndex}:${contentIndex}`;
}

private getOrCreateReasoningState(
  context: CodexStreamContext,
  key: string,
): CodexReasoningState {
  if (!context.reasoningStates.has(key)) {
    context.reasoningStates.set(key, { hasStreamed: false });
  }
  return context.reasoningStates.get(key)!;
}
```

## 使用场景与示例

### 场景 1：简单代码生成

**用户需求**：生成一个 HTTP 服务器

**命令**：
```bash
gal code "用 Node.js 写一个简单的 HTTP 服务器"
```

**流程**：
```
1. CLI 发送请求到网关
   ↓
2. 网关转换格式并调用 Codex
   ↓
3. Codex 返回代码
   data: {"type":"response.output_text.delta","delta":"```javascript\n"}
   data: {"type":"response.output_text.delta","delta":"const http = require('http');\n"}
   data: {"type":"response.output_text.delta","delta":"const server = http.createServer((req, res) => {\n"}
   data: {"type":"response.output_text.delta","delta":"  res.end('Hello World');\n"}
   data: {"type":"response.output_text.delta","delta":"});\n"}
   data: {"type":"response.output_text.delta","delta":"server.listen(3000);\n"}
   data: {"type":"response.output_text.delta","delta":"```"}
   ↓
4. 网关转换并流式返回给 CLI
   ↓
5. CLI 实时显示代码
```

### 场景 2：带推理的复杂问题

**用户需求**：优化算法性能

**配置**：
```yaml
codex:
  reasoning:
    effort: high
  textVerbosity: high
```

**命令**：
```bash
gal code "优化这个排序算法的性能"
```

**流程**：
```
1. Codex 先进行推理
   data: {"type":"response.reasoning_text.delta","delta":"首先分析当前算法..."}
   data: {"type":"response.reasoning_text.delta","delta":"时间复杂度是 O(n²)..."}
   data: {"type":"response.reasoning_text.delta","delta":"可以使用快速排序优化到 O(n log n)..."}
   ↓
2. 然后生成优化后的代码
   data: {"type":"response.output_text.delta","delta":"function quickSort(arr) {\n"}
   data: {"type":"response.output_text.delta","delta":"  if (arr.length <= 1) return arr;\n"}
   ...
   ↓
3. CLI 分别展示推理过程和最终代码
   [思考过程]
   首先分析当前算法...
   时间复杂度是 O(n²)...
   可以使用快速排序优化到 O(n log n)...
   
   [生成代码]
   function quickSort(arr) {
     ...
   }
```

### 场景 3：工具调用集成

**用户需求**：查询并分析天气数据

**工具定义**：
```typescript
const tools = [
  {
    type: 'function',
    function: {
      name: 'get_weather',
      description: '获取城市天气',
      parameters: {
        type: 'object',
        properties: {
          city: { type: 'string', description: '城市名' },
        },
        required: ['city'],
      },
    },
  },
  {
    type: 'function',
    function: {
      name: 'analyze_data',
      description: '分析数据',
      parameters: {
        type: 'object',
        properties: {
          data: { type: 'string', description: '要分析的数据' },
        },
        required: ['data'],
      },
    },
  },
];
```

**命令**：
```bash
gal code --tools tools.json "查询北京天气并分析适合的户外活动"
```

**流程**：
```
1. 模型决定先调用 get_weather
   tool_calls: [{ name: "get_weather", arguments: '{"city":"北京"}' }]
   ↓
2. 客户端执行工具
   result: "晴天，15-25°C，空气质量良"
   ↓
3. 模型决定调用 analyze_data
   tool_calls: [{ name: "analyze_data", arguments: '{"data":"晴天，15-25°C"}' }]
   ↓
4. 客户端执行工具
   result: "温度适中，适合徒步、骑行等户外活动"
   ↓
5. 模型生成最终答案
   "根据分析，北京今天天气晴朗，温度在 15-25°C，非常适合户外活动。
    推荐：徒步、骑行、公园野餐等。"
```

### 场景 4：多轮对话与上下文

**需求**：持续优化代码

**第一轮**：
```bash
gal code "写一个计算斐波那契数列的函数"
```

**Codex 响应**：
```javascript
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

**第二轮**：
```bash
gal code "优化这个函数的性能"
```

**Codex 响应**（保持上下文）：
```javascript
// 使用动态规划优化
function fibonacci(n) {
  const dp = [0, 1];
  for (let i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}
```

**第三轮**：
```bash
gal code "再优化空间复杂度"
```

**Codex 响应**（继续优化）：
```javascript
// 空间复杂度优化到 O(1)
function fibonacci(n) {
  if (n <= 1) return n;
  let prev = 0, curr = 1;
  for (let i = 2; i <= n; i++) {
    [prev, curr] = [curr, prev + curr];
  }
  return curr;
}
```

### 场景 5：OAuth 认证模式

**适用情况**：个人使用，没有 API Key 但有 ChatGPT 账号

**首次配置**：
```bash
gal auth
```

**配置向导**：
```
? 选择 AI Provider: 
  OpenAI
❯ Codex
  Claude Code

? 选择认证模式:
  ApiKey
❯ ChatGPT (OAuth)

配置已保存到 ~/.gemini-any-llm/config.yaml
```

**首次使用**：
```bash
gal code "Hello"
```

**OAuth 登录流程**：
```
正在启动 OAuth 认证...
本地回调服务器已启动: http://127.0.0.1:1455

请在浏览器中打开以下链接完成登录:
https://auth.openai.com/oauth/authorize?client_id=...

等待授权回调...
✓ 授权成功！
✓ Token 已保存到 ~/.gemini-any-llm/codex/auth.json

正在连接 Codex...
```

**后续使用**：
```bash
gal code "写一个函数"
# 自动使用已保存的 Token，无需重新登录
```

## 配置指南

### 基础配置

**全局配置文件**：`~/.gemini-any-llm/config.yaml`

```yaml
# AI 提供商选择
aiProvider: codex

# Codex 配置
codex:
  # 认证模式: ApiKey 或 ChatGPT
  authMode: ApiKey
  
  # API Key (ApiKey 模式必填)
  apiKey: "sk-xxx..."
  
  # API 端点
  baseURL: "https://chatgpt.com/backend-api/codex"
  
  # 模型名称
  model: "gpt-5-codex"
  
  # 请求超时 (毫秒)
  timeout: 1800000
  
  # 推理配置 (可选)
  reasoning:
    effort: medium    # low/medium/high
    summary: auto     # auto/none
  
  # 输出详细度 (可选)
  textVerbosity: medium  # low/medium/high

# 网关配置
gateway:
  port: 23062
  host: "0.0.0.0"
  logLevel: "info"  # debug/info/warn/error
  logDir: "~/.gemini-any-llm/logs"

# 速率限制
rateLimit:
  max: 100          # 最大请求数
  windowMs: 900000  # 时间窗口 (15分钟)

# 请求超时
request:
  timeout: 3600000  # 全局超时 (1小时)

# CORS 配置
allowedOrigins:
  - "http://localhost:3000"
  - "http://127.0.0.1:3000"
```

### 项目特定配置

**项目配置文件**：`./config/config.yaml`

```yaml
# 项目级配置会完全覆盖全局配置
aiProvider: codex

codex:
  authMode: ApiKey
  apiKey: "project-specific-key"
  baseURL: "https://chatgpt.com/backend-api/codex"
  model: "gpt-5-codex"
  timeout: 1800000
  reasoning:
    effort: high      # 项目需要更深度的推理
  textVerbosity: high

gateway:
  port: 23062
  logLevel: "debug"   # 项目开发时使用调试模式
  logDir: "./logs"    # 日志保存在项目目录
```

### 环境变量配置

所有配置都可以通过环境变量覆盖：

```bash
# AI 提供商
export GAL_AI_PROVIDER="codex"

# Codex 配置
export GAL_CODEX_AUTH_MODE="apikey"     # 或 "chatgpt"
export GAL_CODEX_API_KEY="sk-xxx..."
export GAL_CODEX_BASE_URL="https://chatgpt.com/backend-api/codex"
export GAL_CODEX_MODEL="gpt-5-codex"
export GAL_CODEX_TIMEOUT="1800000"

# 推理配置
export GAL_CODEX_REASONING='{"effort":"high","summary":"auto"}'
export GAL_CODEX_TEXT_VERBOSITY="high"

# OAuth 配置
export CODEX_HOME="$HOME/.custom-codex"  # 自定义 Token 目录

# 网关配置
export GAL_PORT="23062"
export GAL_HOST="0.0.0.0"
export GAL_LOG_LEVEL="info"
export GAL_GATEWAY_LOG_DIR="~/.gemini-any-llm/logs"

# 禁用自动更新检查
export GAL_DISABLE_UPDATE_CHECK="1"
```

### 配置优先级

```
环境变量 (最高) > 项目配置 > 全局配置 > 默认值 (最低)
```

**注意**：如果项目配置文件存在，全局配置会被完全忽略（不合并）。

## 故障排查

### 常见问题

#### 1. 认证失败

**现象**：
```
Error: Codex request failed (status 401): Unauthorized
```

**原因**：
- API Key 无效或过期
- OAuth Token 过期且刷新失败

**解决方案**：

**API Key 模式**：
```bash
# 重新配置
gal auth

# 或直接编辑配置文件
vim ~/.gemini-any-llm/config.yaml
# 更新 codex.apiKey
```

**OAuth 模式**：
```bash
# 删除旧 Token
rm ~/.gemini-any-llm/codex/auth.json

# 重新登录
gal code "Hello"
# 会自动触发 OAuth 登录流程
```

#### 2. 回调端口被占用

**现象**：
```
Error: OAuth callback server failed to start: Port 1455 is already in use
```

**原因**：
- 端口 1455 被其他进程占用
- 上次登录的回调服务器未正常关闭

**解决方案**：
```bash
# 方案 1: 查找并终止占用进程
lsof -ti:1455 | xargs kill -9

# 方案 2: 等待自动重试
# ChatGPTAuthManager 会自动尝试多个端口

# 方案 3: 重启 gal 服务
gal restart
```

#### 3. 请求超时

**现象**：
```
Error: Codex request aborted due to timeout
```

**原因**：
- 网络不稳定
- Codex 服务响应慢
- 超时时间设置过短

**解决方案**：
```yaml
# 增加超时时间
codex:
  timeout: 3600000  # 增加到 1 小时
```

#### 4. 推理内容未显示

**现象**：
只看到最终答案，没有推理过程

**原因**：
- 推理配置未启用
- Gemini CLI 版本不支持 reasoning_content

**解决方案**：
```yaml
# 启用推理
codex:
  reasoning:
    effort: medium    # 或 high
  textVerbosity: medium
```

#### 5. 工具调用失败

**现象**：
```
Error: Tool call arguments parse failed
```

**原因**：
- 工具参数 JSON 格式错误
- 工具定义与实际参数不匹配

**解决方案**：
```typescript
// 确保工具定义正确
{
  type: 'function',
  function: {
    name: 'tool_name',
    description: '清晰的描述',
    parameters: {
      type: 'object',
      properties: {
        param: {
          type: 'string',
          description: '参数说明',  // 重要！
        },
      },
      required: ['param'],  // 明确必填参数
    },
  },
}
```

#### 6. 流式响应中断

**现象**：
响应到一半就停止了

**原因**：
- 网络连接中断
- 服务端超时
- 客户端超时

**解决方案**：
```yaml
# 1. 增加超时时间
codex:
  timeout: 3600000

# 2. 检查网络稳定性
# 3. 查看日志
tail -f ~/.gemini-any-llm/logs/gateway-*.log
```

### 调试技巧

#### 1. 启用调试日志

```yaml
gateway:
  logLevel: "debug"
```

或

```bash
export GAL_LOG_LEVEL="debug"
gal restart
```

#### 2. 查看详细日志

```bash
# 实时查看网关日志
tail -f ~/.gemini-any-llm/logs/gateway-*.log

# 查看最近的错误
grep ERROR ~/.gemini-any-llm/logs/gateway-*.log | tail -20

# 查看 Codex 请求
grep "CodexProvider.*outbound" ~/.gemini-any-llm/logs/gateway-*.log

# 查看 Codex 响应
grep "CodexProvider.*inbound" ~/.gemini-any-llm/logs/gateway-*.log
```

#### 3. 测试认证

```bash
# API Key 模式
curl -X POST https://chatgpt.com/backend-api/codex/responses \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-5-codex","instructions":"Test","input":[],"stream":false}'

# OAuth 模式 - 检查 Token
cat ~/.gemini-any-llm/codex/auth.json | jq .
```

#### 4. 检查服务状态

```bash
# 查看网关状态
gal status

# 查看进程
ps aux | grep gal

# 查看端口占用
lsof -i:23062
```

#### 5. 重置环境

```bash
# 完全重置
gal kill
rm -rf ~/.gemini-any-llm/codex/
rm ~/.gemini-any-llm/config.yaml
gal auth
```

### 性能优化

#### 1. 使用提示缓存

Codex Provider 自动使用 `prompt_cache_key` 来缓存提示词：

```typescript
const payload: CodexRequest = {
  // ...
  prompt_cache_key: this.conversationId,  // 同一会话使用相同的 key
};
```

**效果**：
- 减少重复发送系统提示词
- 加快响应速度
- 降低 Token 消耗

#### 2. 调整推理强度

根据任务复杂度选择合适的推理强度：

```yaml
codex:
  reasoning:
    effort: low     # 简单任务，快速响应
    # effort: medium  # 平衡模式
    # effort: high    # 复杂任务，深度思考
```

#### 3. 控制输出详细度

```yaml
codex:
  textVerbosity: low     # 简洁输出，适合代码生成
  # textVerbosity: medium  # 标准输出
  # textVerbosity: high    # 详细解释，适合教学
```

#### 4. 并发控制

```yaml
rateLimit:
  max: 50         # 降低并发数可以提高稳定性
  windowMs: 900000
```

## 总结

### 核心特性回顾

1. **双认证模式**：支持 API Key 和 OAuth，灵活适应不同场景
2. **协议转换**：自动处理 Gemini ↔ Codex 格式转换
3. **流式响应**：高性能 SSE 流式输出，实时展示结果
4. **工具调用**：完整的 Function Calling 支持
5. **推理展示**：展示模型思考过程，提高可解释性
6. **自动刷新**：OAuth Token 自动刷新，无需手动维护

### 最佳实践

1. **选择合适的认证模式**
   - 团队/企业：使用 API Key 模式
   - 个人：使用 OAuth 模式

2. **合理配置推理参数**
   - 简单任务：`effort: low`, `textVerbosity: low`
   - 复杂任务：`effort: high`, `textVerbosity: high`

3. **使用项目配置**
   - 不同项目使用不同配置
   - 避免修改全局配置

4. **监控日志**
   - 生产环境使用 `info` 级别
   - 开发环境使用 `debug` 级别

5. **定期更新**
   - 使用 `gal update` 获取最新功能
   - 关注 Token 过期时间

### 参考资源

- **项目仓库**：https://github.com/x22x22/gemini-any-llm
- **开发文档**：[DEVELOPMENT.md](../DEVELOPMENT.md)
- **架构文档**：[CLAUDE.md](../CLAUDE.md)
- **配置示例**：[config/config.example.yaml](../config/config.example.yaml)
- **Gemini CLI**：https://github.com/google/gemini-cli

---

**文档版本**：1.0.0  
**最后更新**：2025-10-30  
**维护者**：gemini-any-llm 团队
