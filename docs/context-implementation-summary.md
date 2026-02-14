# OpenClaw 上下文实现机制总结

本文档总结 OpenClaw Agent 的上下文管理实现思路，涵盖系统提示词、会话历史、记忆系统、工具系统等核心组件的设计与协作方式。

---

## 目录

1. [设计理念](#1-设计理念)
2. [上下文组成结构](#2-上下文组成结构)
3. [系统提示词组装](#3-系统提示词组装)
4. [会话历史管理](#4-会话历史管理)
5. [记忆系统](#5-记忆系统)
6. [工具系统](#6-工具系统)
7. [上下文压缩](#7-上下文压缩)
8. [完整数据流](#8-完整数据流)
9. [用户交互流程](#9-用户交互流程)
10. [媒体理解](#10-媒体理解)
11. [模型 API 接口](#11-模型-api-接口)
12. [关键设计决策](#12-关键设计决策)

---

## 1. 设计理念

OpenClaw 的上下文管理遵循以下核心原则：

### 1.1 持久化与发送解耦

```
JSONL 文件 = 完整的历史归档（append-only，用于审计/调试）
session.messages = 经过处理的"可发送版本"（按需裁剪）
```

每次请求前，从持久化层加载 → 清洗 → 验证 → 限制 → 发送。这样既保留完整记录，又能灵活控制发送内容。

### 1.2 模块化拼接

系统提示词不是静态模板，而是根据当前状态动态组装：
- 工具列表（按策略过滤后的）
- 技能配置
- 工作区文件（SOUL.md、MEMORY.md 等）
- 运行时信息（模型、渠道、沙盒状态）

### 1.3 多层适配

同一套历史数据，通过适配层支持不同 LLM Provider：
- Anthropic：thinking block 签名格式
- Google：轮次顺序要求（不能以 assistant 开头）
- OpenAI：reasoning block 降级

### 1.4 容错优先

系统设计为"可恢复"：
- 缺失的 toolResult → 自动合成
- 格式错误的消息 → 清洗修复
- 上下文溢出 → 自动压缩

---

## 2. 上下文组成结构

发送给 LLM 的完整上下文由三部分组成：

```
┌─────────────────────────────────────────────────────────────┐
│  System Prompt（系统提示词）                                 │
│  ├── 身份声明                                               │
│  ├── 工具列表 + 描述                                        │
│  ├── 安全约束                                               │
│  ├── 技能指令                                               │
│  ├── 记忆召回指令                                           │
│  ├── 运行时信息                                             │
│  └── Project Context（SOUL.md、MEMORY.md 等文件内容）       │
├─────────────────────────────────────────────────────────────┤
│  Messages（会话历史）                                        │
│  ├── [summary]（压缩后的摘要，如果有）                       │
│  ├── user: "..."                                            │
│  ├── assistant: "..." + [toolUse]                           │
│  ├── toolResult: "..."                                      │
│  ├── assistant: "..."                                       │
│  └── user: "当前消息"                                       │
├─────────────────────────────────────────────────────────────┤
│  Tools（工具定义）                                           │
│  ├── name + description + parameters (JSON Schema)          │
│  └── 按 provider 标准化后的格式                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 系统提示词组装

### 3.1 组装入口

```
attempt.ts:344 → buildEmbeddedSystemPrompt()
                      ↓
              buildAgentSystemPrompt()  ← 核心拼接函数
```

### 3.2 段落结构

系统提示词按顺序包含以下段落（可按条件跳过）：

| 段落 | 构建函数 | 条件 |
|------|----------|------|
| 身份声明 | 硬编码 | 始终 |
| ## Tooling | 内联 | 始终 |
| ## Tool Call Style | 内联 | 始终 |
| ## Safety | `buildSafetySection()` | 始终 |
| ## Skills | `buildSkillsSection()` | 非 minimal 模式 |
| ## Memory Recall | `buildMemorySection()` | 有 memory 工具 |
| ## Workspace | 内联 | 始终 |
| ## Sandbox | 内联 | 沙盒启用时 |
| ## User Identity | `buildUserIdentitySection()` | 有 owner 号码 |
| ## Reply Tags | `buildReplyTagsSection()` | 非 minimal |
| ## Messaging | `buildMessagingSection()` | 有 message 工具 |
| # Project Context | 内联 | 有 context files |
| ## Silent Replies | 内联 | 非 minimal |
| ## Heartbeats | 内联 | 非 minimal |
| ## Runtime | `buildRuntimeLine()` | 始终 |

### 3.3 Bootstrap 文件注入

工作区中的 `.md` 文件会被读取并注入到 `# Project Context` 段落：

```
扫描顺序：
1. AGENTS.md    ← 技能、子 Agent 配置
2. SOUL.md      ← 人格、语气
3. TOOLS.md     ← 外部工具使用指南
4. IDENTITY.md  ← 身份信息
5. USER.md      ← 用户偏好
6. HEARTBEAT.md ← 心跳提示词
7. BOOTSTRAP.md ← 通用引导
8. MEMORY.md / memory.md  ← 记忆文件（可选）
```

每个文件最多 20,000 字符，超长时保留头部 70% + 尾部 20%，中间插入截断标记。

### 3.4 三种模式

| 模式 | 使用场景 | 包含段落 |
|------|----------|----------|
| `full` | 主 Agent | 全部 |
| `minimal` | 子 Agent | 仅 Tooling、Safety、Workspace、Sandbox、Runtime |
| `none` | 极简 | 仅身份声明一句话 |

---

## 4. 会话历史管理

### 4.1 存储格式

每个会话对应一个 JSONL 文件：

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

三种 entry 类型：

```jsonl
{"type":"session","version":2,"id":"...","cwd":"..."}
{"type":"message","message":{"role":"user","content":"..."}}
{"type":"custom","customType":"model-snapshot","data":{...}}
```

### 4.2 并发安全

写入前必须获取文件锁：

```typescript
// O_EXCL 原子创建锁文件
const handle = await fs.open(lockPath, "wx");
// 写入 PID + 时间戳
// 超时 30 分钟或进程死亡时自动清理
```

### 4.3 消息写入

`appendMessage` 被 monkey-patch，添加两个功能：

1. **工具配对检查**：确保每个 `toolUse` 都有对应的 `toolResult`，缺失时合成
2. **事件广播**：通知 Memory 系统更新索引

### 4.4 加载时清洗

每次发送前，历史消息经过多层处理：

```
原始 messages[]
  ↓ sanitizeSessionMessagesImages()     ← 图片清理
  ↓ sanitizeAntigravityThinkingBlocks() ← thinking block 清理
  ↓ sanitizeToolUseResultPairing()      ← 工具配对修复
  ↓ downgradeOpenAIReasoningBlocks()    ← reasoning 降级
  ↓ validateGeminiTurns()               ← 轮次验证
  ↓ validateAnthropicTurns()            ← 轮次验证
  ↓ limitHistoryTurns()                 ← 历史限制
  ↓
最终 messages[]
```

### 4.5 历史限制

DM 会话可配置只保留最近 N 轮：

```yaml
channels:
  telegram:
    dmHistoryLimit: 50
```

---

## 5. 记忆系统

### 5.1 双层存储

| 层 | 存储位置 | 内容 | 用途 |
|----|----------|------|------|
| 文件层 | MEMORY.md, memory/*.md | 持久化记忆 | 长期存储 |
| 索引层 | SQLite (chunks, chunks_vec, chunks_fts) | 向量 + 全文索引 | 快速检索 |

### 5.2 索引机制

```
Memory 文件变化（chokidar 监控）
  ↓
listMemoryFiles() 扫描 .md 文件
  ↓
hash 比较（SHA-256）
  ↓ 变化的文件
chunkMarkdown()  ← 分块（~1600 字符/块，320 字符重叠）
  ↓
generateEmbedding()  ← 生成向量
  ↓
写入 SQLite 三张表
  ├── chunks      ← 原文分块
  ├── chunks_vec  ← 向量索引 (sqlite-vec)
  └── chunks_fts  ← 全文索引 (FTS5)
```

### 5.3 混合搜索

```
最终分数 = 0.7 × 向量相似度 + 0.3 × BM25 文本分数
```

### 5.4 Memory Flush

在上下文压缩前，系统会给 Agent 一个机会保存重要信息：

```
触发条件：totalTokens >= contextWindow - reserve - 4000

执行：启动一轮特殊 Agent 对话
Prompt："Store durable memories now (use memory/YYYY-MM-DD.md)"

Agent 自主决定：
  ├── 有重要信息 → 用 write 工具写入 memory/*.md
  └── 没什么重要的 → 返回 SILENT_REPLY_TOKEN
```

---

## 6. 工具系统

### 6.1 工具组装流水线

```
createOpenClawCodingTools()
  ↓
步骤1: 基础编码工具 (pi-coding-agent)
步骤2: 执行工具 (exec, process)
步骤3: 渠道工具 (message)
步骤4: OpenClaw 工具 (browser, canvas, cron, memory)
步骤5: 七层策略过滤
步骤6: Schema 标准化（按 provider）
步骤7: 生命周期钩子（abort 信号）
```

### 6.2 七层策略过滤

工具可用性经过层层过滤：

```
Profile → Provider → Global → Agent → Group → Sandbox → Sub-agent
```

### 6.3 Schema 适配

不同 provider 对 JSON Schema 支持不同：

| Provider | 限制 | 处理 |
|----------|------|------|
| OpenAI | 不支持 anyOf/oneOf | 展平为单一类型 |
| Gemini | 不认识 format 等关键字 | 清理掉 |
| Anthropic | 支持最完整 | 无需处理 |

---

## 7. 上下文压缩

### 7.1 触发条件

```
每轮 LLM 响应后检查：
  totalTokens > contextWindow - reserveTokens?
    ├── 是 → 执行压缩
    └── 否 → 继续
```

### 7.2 压缩算法

```
summarizeInStages()

1. 按 token 比例分段（BASE_CHUNK_RATIO = 0.4）
2. 每段独立调用 LLM 生成摘要
3. 合并所有摘要
4. 安全系数放大（SAFETY_MARGIN = 1.2）

压缩后结构：
[summary message] + [recent uncompressed messages]
```

### 7.3 Memory Flush + Compaction 协作

```
上下文快满
  ↓
Memory Flush（给 Agent 机会保存重要信息到 memory/*.md）
  ↓
Compaction（压缩历史为摘要）
  ↓
继续对话
```

这样即使对话被压缩，重要信息仍可通过 `memory_search` 检索到。

---

## 8. 完整数据流

### 8.1 请求组装流程

```
用户消息到达
  ↓
路由 → 确定 Session Key
  ↓
acquireSessionWriteLock()  ← 获取写锁
  ↓
SessionManager.open()  ← 加载 JSONL
  ↓
sanitizeSessionHistory()  ← 多层清洗
  ↓
limitHistoryTurns()  ← 历史限制
  ↓
session.agent.replaceMessages()  ← 替换内存消息
  ↓
buildEmbeddedSystemPrompt()  ← 组装系统提示词
  ├── 加载 bootstrap 文件
  ├── 组装工具列表
  ├── 拼接各段落
  └── 注入 Project Context
  ↓
createOpenClawCodingTools()  ← 组装工具集
  ├── 七层策略过滤
  └── Schema 标准化
  ↓
streamSimple()  ← 发送到 LLM
  ↓
处理响应
  ├── 文本 → 输出
  └── 工具调用 → 执行 → toolResult → 继续循环
  ↓
guardedAppend()  ← 写入 JSONL
  ↓
emitSessionTranscriptUpdate()  ← 通知 Memory 系统
  ↓
检查是否需要压缩
  ├── 需要 → Memory Flush → Compaction → 重新循环
  └── 不需要 → 完成
  ↓
sessionLock.release()  ← 释放写锁
```

### 8.2 记忆同步流程

```
Agent 写入 memory/*.md
  ↓
chokidar 检测变化 → dirty 标志
  ↓
下次 sync → hash 比较 → 分块 → embedding → SQLite
  ↓
后续对话通过 memory_search 检索
```

---

## 9. 用户交互流程

### 10.1 多层架构

```
用户层（各平台客户端）
        │
        ↓
渠道适配层（telegram/bot.ts, discord/bot.ts, ...）
  - 接收原始消息
  - 下载媒体文件
  - 构建 MsgContext
        │
        ↓
Gateway 网关层
  - 路由决策（哪个 Agent 处理）
  - 权限检查（allowFrom、群组策略）
  - 会话管理（SessionKey 生成）
        │
        ↓
Auto-Reply 处理层
  - 媒体理解（图片/音频/视频转文字）
  - 链接理解（URL 抓取摘要）
  - 指令解析（/model、/reset）
  - 触发 Agent 执行
        │
        ↓
Agent 执行层
  - 工具调用
  - 流式输出
        │
        ↓
Reply Dispatcher 回复分发层
  - 工具结果回复 (tool)
  - 块级回复 (block)
  - 最终回复 (final)
        │
        ↓
渠道投递层
  - 消息分块
  - 格式转换
  - 媒体上传
```

### 10.2 消息上下文（MsgContext）

每条入站消息被封装为统一的 `MsgContext` 对象：

| 字段 | 说明 |
|------|------|
| `Body` | 原始消息体 |
| `BodyForAgent` | 给 Agent 的格式化消息（含历史上下文） |
| `From` / `SenderName` | 发送者信息 |
| `SessionKey` | 会话路由键 |
| `Provider` | 渠道类型（telegram/discord/...） |
| `ChatType` | 对话类型（private/group） |
| `WasMentioned` | 是否被 @ |
| `MediaPath` / `MediaType` | 媒体文件信息 |
| `MediaUnderstanding` | 媒体理解结果 |

### 9.3 Reply Dispatcher

回复分发器串行化投递，支持三种回复类型：

```typescript
const dispatcher = createReplyDispatcher({
    deliver: async (payload, { kind }) => {
        // kind: "tool" | "block" | "final"
        await sendToChannel(payload);
    },
    humanDelay: { mode: "default" },  // 可选人类延迟
});

// Agent 执行过程中调用
dispatcher.sendToolResult({ body: "文件已创建" });
dispatcher.sendBlockReply({ body: "正在处理..." });
dispatcher.sendFinalReply({ body: "完成！" });
```

---

## 10. 媒体理解

### 10.1 三种能力

| 能力 | 输出类型 | 用途 |
|------|----------|------|
| `image` | `image.description` | 图片内容描述 |
| `audio` | `audio.transcription` | 音频转文字 |
| `video` | `video.description` | 视频内容描述 |

### 10.2 两种处理模式

**Provider 模式（调用 API）**：

| Provider | 图片 | 音频 | 视频 |
|----------|------|------|------|
| OpenAI | ✅ gpt-5-mini | ✅ whisper-1 | ❌ |
| Anthropic | ✅ claude-opus-4-5 | ❌ | ❌ |
| Google | ✅ gemini-3-flash | ✅ | ✅ |
| DeepGram | ❌ | ✅ | ❌ |
| Groq | ❌ | ✅ whisper-large-v3 | ❌ |

**CLI 模式（本地命令）**：

| 工具 | 用途 |
|------|------|
| `whisper` | OpenAI Whisper 本地版 |
| `whisper-cli` | whisper.cpp |
| `sherpa-onnx-offline` | 离线语音识别 |
| `gemini` | Gemini CLI |

### 10.3 自动 Provider 选择

```
1. 优先使用当前对话的模型（如果支持该能力）
2. 音频：优先本地工具（whisper/sherpa-onnx）
3. 检查 Gemini CLI
4. 按优先级检查 API Key（openai → anthropic → google → ...）
```

### 10.4 智能跳过

当主模型本身支持视觉能力时（如 GPT-4o），跳过图片预处理，直接把图片发给模型。

---

## 11. 模型 API 接口

### 11.1 支持的 API 类型

| API 类型 | 对应 Provider | 说明 |
|----------|--------------|------|
| `anthropic-messages` | Anthropic (Claude) | Anthropic Messages API |
| `openai-completions` | OpenAI, 兼容服务 | OpenAI Chat Completions API |
| `openai-responses` | OpenAI o1/o3 | OpenAI Responses API (推理模型) |
| `google-generative-ai` | Google Gemini | Google Generative AI SDK |
| `bedrock-converse-stream` | AWS Bedrock | AWS Bedrock Converse API |

### 11.2 OpenAI 兼容服务

使用 `openai-completions` API 的第三方服务：

- MiniMax (`https://api.minimax.chat/v1`)
- Moonshot/Kimi (`https://api.moonshot.ai/v1`)
- Qwen Portal (`https://portal.qwen.ai/v1`)
- Ollama (`http://127.0.0.1:11434/v1`)
- Venice, OpenRouter, GitHub Copilot

### 11.3 历史适配（TranscriptPolicy）

不同 API 对消息格式有不同要求：

| 策略 | OpenAI | Anthropic | Google |
|------|--------|-----------|--------|
| `sanitizeToolCallIds` | ❌ | ❌ | ✅ |
| `repairToolUseResultPairing` | ❌ | ✅ | ✅ |
| `validateTurns` | ❌ | ✅ | ✅ |
| `applyGoogleTurnOrdering` | ❌ | ❌ | ✅ |

---

## 12. 关键设计决策

### 9.1 为什么用 JSONL 而不是数据库？

- **简单可靠**：纯文本，无需数据库依赖
- **Append-only**：自然的时间序列，便于追溯
- **可读性强**：直接用文本编辑器查看调试
- **跨平台**：无二进制兼容问题

### 9.2 为什么要多层清洗？

- **跨模型兼容**：同一会话可在不同 provider 间切换
- **容错恢复**：历史可能被中断（崩溃、超时），需要修复
- **格式演进**：旧格式消息需要适配新 API

### 9.3 为什么 Memory 用 SQLite + 向量索引？

- **混合搜索**：语义相似（向量）+ 关键词匹配（FTS）
- **本地优先**：无需外部服务，隐私友好
- **增量更新**：hash 比较避免重复索引

### 9.4 为什么系统提示词动态组装？

- **状态感知**：工具可用性、沙盒状态、渠道能力都可能变化
- **模式切换**：主 Agent 和子 Agent 需要不同的提示词
- **个性化**：SOUL.md、IDENTITY.md 等文件允许用户定制

### 9.5 为什么分离 Memory Flush 和 Compaction？

- **职责分离**：Compaction 是机械的摘要，Memory Flush 是智能的筛选
- **信息保留**：Compaction 丢失细节，Memory Flush 保留 Agent 认为重要的信息
- **可检索**：memory/*.md 被索引，压缩后仍可通过 memory_search 找回

---

## 关键源码文件

| 文件 | 职责 |
|------|------|
| `src/agents/system-prompt.ts` | 系统提示词组装 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Agent 单次执行 |
| `src/agents/pi-tools.ts` | 工具组装流水线 |
| `src/agents/session-write-lock.ts` | 会话文件写锁 |
| `src/agents/session-tool-result-guard.ts` | 消息写入包装 |
| `src/agents/pi-embedded-runner/google.ts` | 历史清洗 |
| `src/agents/pi-embedded-runner/history.ts` | 历史限制 |
| `src/agents/compaction.ts` | 压缩算法 |
| `src/memory/manager.ts` | 记忆索引管理 |
| `src/memory/hybrid.ts` | 混合搜索 |
| `src/agents/workspace.ts` | Bootstrap 文件加载 |
| `src/auto-reply/reply/memory-flush.ts` | Memory Flush 触发 |

---

## 总结

OpenClaw 的上下文管理体现了"工程化"的思路：

1. **不是简单的 messages 传递**，而是完整的数据管道（加载→清洗→限制→发送）
2. **不是单一的历史存储**，而是多层结构（JSONL 归档 + SQLite 索引 + 内存缓存）
3. **不是静态的提示词**，而是动态组装（按状态、模式、文件内容拼接）
4. **不是被动的上下文管理**，而是主动的容量控制（压缩 + 限制 + Memory Flush）

这套设计使得 OpenClaw 能够：
- 支持长期运行的会话而不溢出
- 在不同 LLM Provider 间无缝切换
- 保留完整历史用于调试同时优化发送内容
- 通过记忆系统实现跨会话的信息持久化
