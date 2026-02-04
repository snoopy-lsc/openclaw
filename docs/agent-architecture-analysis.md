# OpenClaw Agent 架构深度分析

本文档对 OpenClaw 项目的 Agent 实现进行全面拆解，面向初学者，逐层分析从入口到底层的完整架构。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [Agent 执行循环](#2-agent-执行循环)
3. [工具系统](#3-工具系统)
4. [会话与记忆系统](#4-会话与记忆系统)
5. [MEMORY.md 管理机制](#5-memorymd-管理机制)
6. [关键数据流图](#6-关键数据流图)

---

## 1. 整体架构概览

### 1.1 核心依赖

OpenClaw 的 Agent 实现基于三个外部核心库：

| 库 | 职责 |
|---|---|
| `@mariozechner/pi-agent-core` | Agent 执行引擎核心（循环、工具调用、消息管理） |
| `@mariozechner/pi-ai` | LLM 提供者抽象层（OpenAI / Anthropic / Google 等） |
| `@mariozechner/pi-coding-agent` | 编码工具集（文件读写、搜索、Bash 执行等） |

### 1.2 入口点

Agent 有三个主要入口：

```
CLI（命令行）        → src/commands/agent.ts
Gateway（网关/API）  → src/auto-reply/reply/agent-runner.ts
Cron（定时任务）     → src/auto-reply/heartbeat.ts
```

所有入口最终都调用同一个核心函数：

```
runEmbeddedPiAgent()  →  src/agents/pi-embedded-runner/run.ts
```

### 1.3 分层架构

```
┌────────────────────────────────────┐
│  入口层 (CLI / Gateway / Cron)     │
├────────────────────────────────────┤
│  模型选择 & 重试 (model-fallback)  │
├────────────────────────────────────┤
│  执行引擎 (runEmbeddedPiAgent)     │
│    ├── 会话管理 (SessionManager)   │
│    ├── 工具组装 (pi-tools)         │
│    ├── 系统提示词 (system-prompt)  │
│    └── 沙盒策略 (sandbox)          │
├────────────────────────────────────┤
│  LLM 调用 (streamSimple)          │
├────────────────────────────────────┤
│  持久化层                          │
│    ├── Session Store (JSON)        │
│    ├── Transcript (JSONL)          │
│    └── Memory Index (SQLite)       │
└────────────────────────────────────┘
```

---

## 2. Agent 执行循环

### 2.1 主循环 (`runEmbeddedPiAgent`)

文件：`src/agents/pi-embedded-runner/run.ts`

```
while (true) {
  1. 准备沙盒环境、技能快照
  2. 组装工具集
  3. 构建系统提示词
  4. 创建 SessionManager（加载历史）
  5. 调用 streamSimple()（发送到 LLM）
  6. 处理响应（工具调用 or 文本回复）
  7. 检查是否需要 compaction（上下文压缩）
  8. 如需 compaction → 执行压缩 → 回到步骤 1
  9. 如果完成 → 退出循环
}
```

### 2.2 单次尝试 (`runEmbeddedAttempt`)

文件：`src/agents/pi-embedded-runner/run/attempt.ts`

每次循环迭代的核心步骤：

1. **沙盒检查** — 确定工作目录的读写权限
2. **技能加载** — 从 `AGENTS.md`、插件等加载 Agent 的技能配置
3. **工具组装** — 调用 `createOpenClawCodingTools()` 生成完整工具集
4. **系统提示词** — 组合身份、技能、上下文、记忆等段落
5. **SessionManager** — 加载历史消息、管理上下文窗口
6. **`streamSimple()`** — 将消息发送给 LLM，流式处理响应

### 2.3 错误处理与重试

- **模型降级**：主模型失败时自动切换到备用模型（`model-fallback.ts`）
- **Compaction 重试**：上下文超限时压缩历史并重新尝试
- **工具执行超时**：每个工具有独立的超时和 abort 信号

---

## 3. 工具系统

### 3.1 工具接口

每个工具都实现 `AnyAgentTool` 接口：

```typescript
interface AnyAgentTool {
  name: string;                    // 工具名称
  description: string;             // 给 LLM 看的描述
  parameters: JSONSchema;          // 参数的 JSON Schema
  execute(params): Promise<Result>; // 执行函数
}
```

执行结果类型：

```typescript
type ToolResult =
  | string                          // 纯文本
  | { text: string }                // 文本对象
  | { image: Buffer, mimeType }     // 图片
  | { parts: ToolResultPart[] }     // 混合内容
```

### 3.2 工具组装流水线

文件：`src/agents/pi-tools.ts` — `createOpenClawCodingTools()`

```
步骤1: 基础编码工具 (pi-coding-agent)
  → 文件读写、搜索、Bash 执行、Glob 等
  ↓
步骤2: 执行工具 (bash-tools.ts)
  → exec、process 管理工具
  ↓
步骤3: 渠道工具 (channel tools)
  → 消息发送、回复等（按渠道不同）
  ↓
步骤4: OpenClaw 自有工具 (openclaw-tools.ts)
  → 浏览器、画布、定时任务、图片、TTS、记忆搜索等
  ↓
步骤5: 7 层策略过滤
  → 决定哪些工具可用
  ↓
步骤6: Schema 标准化
  → 兼容不同 LLM 提供者
  ↓
步骤7: 生命周期钩子
  → abort 信号包装、执行前后回调
```

### 3.3 七层工具策略过滤

工具的可用性经过七层策略过滤，从外到内：

| 层级 | 来源 | 作用 |
|------|------|------|
| 1. Profile | 用户配置文件 | 全局工具白/黑名单 |
| 2. Provider | LLM 提供者 | 不同模型支持的工具差异 |
| 3. Global | 全局配置 | 系统级工具策略 |
| 4. Agent | Agent 配置 | 单个 Agent 的工具权限 |
| 5. Group | 群组配置 | 群聊场景的工具限制 |
| 6. Sandbox | 沙盒策略 | 安全隔离（只读/读写） |
| 7. Sub-agent | 子 Agent | 父 Agent 限制子 Agent 的工具 |

### 3.4 Schema 标准化

文件：`src/agents/pi-tools.schema.ts`

不同 LLM 提供者对 JSON Schema 的支持不同：

- **OpenAI**：不支持 `anyOf` / `oneOf`，需要展平为单一类型
- **Gemini**：不认识某些关键字（如 `format`），需要清理
- **Claude**：支持最完整的 Schema

`normalizeToolParameters()` 会根据目标提供者自动转换 Schema 格式。

### 3.5 参数兼容层

不同版本的工具可能使用不同的参数名。系统通过参数映射保持向后兼容：

```
file_path ↔ path
old_string ↔ oldText
new_string ↔ newText
```

### 3.6 具体工具举例

**Browser Tool** (`src/agents/tools/browser-tool.ts`):
- 基于 Playwright 的浏览器自动化
- 三种运行模式：沙盒内 / 宿主机 / 远程节点
- 支持截图、点击、输入、导航等操作

**Memory Tool** (`src/agents/tools/memory-tool.ts`):
- `memory_search`：向量 + 关键词混合搜索记忆
- `memory_get`：直接读取指定记忆文件

**Plugin Tools** (`src/plugins/tools.ts`):
- 从 `extensions/` 目录加载插件提供的工具
- 名称冲突检测（插件工具不能覆盖核心工具）
- 可选的工具白名单过滤

---

## 4. 会话与记忆系统

### 4.1 会话 Key 路由

每个对话都有唯一的 Session Key，格式：

```
agent:{agentId}:{scope}:{identifier}
```

不同场景的 Key 示例：

| 场景 | Key 格式 |
|------|----------|
| 私聊 | `agent:default:dm:+1234567890` |
| 群聊 | `agent:default:group:chat-id-123` |
| 话题 | `agent:default:thread:thread-id-456` |
| CLI | `agent:default:cli:session-name` |

### 4.2 Session Store（会话存储）

文件：`src/config/sessions/store.ts`

- **存储格式**：JSON 文件（`sessions.json`）
- **缓存**：45 秒 TTL 内存缓存，避免频繁磁盘读取
- **并发安全**：
  - 文件锁（`.lock` 文件，`O_EXCL` 原子创建）
  - 读在锁内执行（read-inside-lock 模式）
  - 原子写入（先写 tmp 文件，再 rename）
- **SessionEntry** 包含约 70 个字段（token 计数、压缩次数、模型信息等）

### 4.3 Transcript（对话记录）

文件：`src/config/sessions/transcript.ts`

- **格式**：JSONL（每行一个 JSON 对象）
- **内容**：Session Header + 逐条消息记录
- **触发**：每次 `sessionManager.appendMessage` 后通过事件总线广播

```jsonl
{"type":"session","agentId":"default","model":"gpt-4",...}
{"type":"message","role":"user","content":[{"type":"text","text":"Hello"}]}
{"type":"message","role":"assistant","content":[{"type":"text","text":"Hi!"}]}
```

### 4.4 Compaction（上下文压缩）

文件：`src/agents/compaction.ts`

当对话超出上下文窗口时，自动执行压缩：

```
算法：summarizeInStages()

1. 按 token 比例分段（BASE_CHUNK_RATIO = 0.4）
2. 每段独立调用 LLM 做摘要
3. 合并所有摘要
4. 安全系数放大（SAFETY_MARGIN = 1.2）

备选方案：
- 如果某条消息过长 → 截断后摘要（summarizeWithFallback）
- 如果全部摘要仍超限 → 递归再次压缩
```

### 4.5 Memory 向量索引

文件：`src/memory/manager.ts`、`src/memory/memory-schema.ts`

SQLite 数据库中有以下表：

| 表名 | 类型 | 用途 |
|------|------|------|
| `chunks` | 普通表 | 存储文本分块及元数据 |
| `chunks_vec` | 虚拟表 (sqlite-vec) | 向量索引（embedding 相似度搜索） |
| `chunks_fts` | 虚拟表 (FTS5) | 全文搜索索引 |
| `embedding_cache` | 普通表 | embedding 缓存（避免重复计算） |
| `files` | 普通表 | 文件 hash 记录（变更检测） |
| `meta` | 普通表 | 索引元数据 |

### 4.6 混合搜索

文件：`src/memory/hybrid.ts`

```
最终分数 = 0.7 × 向量相似度分数 + 0.3 × BM25 文本分数
```

- 向量搜索：基于 embedding 的余弦相似度
- 文本搜索：FTS5 的 BM25 排名，转换公式 `score = 1 / (1 + rank)`
- 结果按 chunk ID 合并，加权求和后排序

### 4.7 内容选择：什么会被索引到 SQLite

#### 两道主门

| 门 | 判断 | 来源 |
|----|------|------|
| Memory 文件 | MEMORY.md、memory.md、memory/ 目录下的 `.md` 文件 | `listMemoryFiles()` |
| Session 文件 | 配置中启用了 `sources: ["sessions"]` 才索引 | `shouldSyncSessions()` |

#### 三道小门（内容过滤）

1. **Hash 去重**：文件内容未变化（SHA-256 相同）则跳过
2. **内容提取**：Session JSONL 只提取 `role=user/assistant` 且 `type=text` 的内容块
3. **空块过滤**：分块后空白块被丢弃

#### 分块算法

```
每块约 1600 字符（400 tokens × 4 chars/token）
重叠 320 字符（80 tokens × 4）
按行边界对齐（不会切断一行的中间）
```

### 4.8 Dirty 列表机制（增量同步）

Session 文件的变更检测不依赖文件监控，而是通过事件链：

```
Agent 写入消息 (appendMessage)
  ↓
installSessionToolResultGuard 拦截
  ↓
emitSessionTranscriptUpdate(sessionFile)  ← 事件发射
  ↓
MemoryIndexManager.ensureSessionListener()  ← 事件订阅
  ↓
pendingFiles.add(sessionFile)  ← 加入待处理集合
  ↓
5 秒防抖 (debounce)
  ↓
processSessionDeltaBatch()  ← 批量检查
  ↓
检查 delta 阈值：变化 > 100KB 或 > 50 条消息？
  ├── 是 → dirtyFiles.add(file) → 等待下次 sync
  └── 否 → 继续累积，等下次触发
```

---

## 5. MEMORY.md 管理机制

### 5.1 创建方式

MEMORY.md **不是系统自动创建的**（没有模板文件）。它由 Agent 自己通过 `write` 工具创建和写入。

### 5.2 Memory Flush（记忆刷盘）

文件：`src/auto-reply/reply/memory-flush.ts`、`src/auto-reply/reply/agent-runner-memory.ts`

**触发条件**：

```
totalTokens >= contextWindow - reserveTokens - 4000
且本次 compaction 周期内还没执行过 flush
```

**执行过程**：

1. 系统在 compaction 之前启动一轮**特殊 Agent 对话**
2. 发送 prompt："Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed)."
3. Agent **自主判断**哪些信息值得长期保留
4. 值得保留 → Agent 用 `write` 工具写入 `memory/YYYY-MM-DD.md`
5. 没什么重要的 → Agent 返回 `SILENT_REPLY_TOKEN`，什么都不写
6. 更新 session store 的 `memoryFlushCompactionCount`（防止重复触发）

**关键设计**：系统提示词说 "usually SILENT_REPLY_TOKEN is correct"，即大多数情况下 Agent 认为不需要写任何东西。只有真正重要的信息才会被持久化。

### 5.3 Bootstrap 注入

文件：`src/agents/workspace.ts`

如果 MEMORY.md 存在，它会被作为 bootstrap 文件注入到系统提示词中，Agent 每轮对话都能"看到"它的内容。

### 5.4 系统提示词指令

文件：`src/agents/system-prompt.ts`

系统提示词中包含：

> "Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search on MEMORY.md + memory/*.md"

指示 Agent 在回答历史相关问题前，先搜索记忆文件。

### 5.5 文件监控与索引

- **Chokidar 监控**：`MemoryIndexManager.ensureWatcher()` 监控 MEMORY.md 和 memory/ 目录
- 文件变化 → dirty 标志 → 下次 sync 时重新索引到 SQLite
- 索引过程：分块 → 生成 embedding → 写入三张表

### 5.6 Memory Flush vs Compaction

| 机制 | 执行者 | 写入什么 | 目的 |
|------|--------|----------|------|
| Memory Flush | Agent（LLM） | Agent 认为值得长期保留的摘要 | 在压缩前抢救重要信息 |
| Compaction | 算法 | 对话段落的自动摘要 | 释放上下文窗口空间 |

两者互补：Compaction 保证上下文不溢出但丢失细节，Memory Flush 在压缩前让 Agent 把重要信息存到磁盘文件中。

---

## 6. 关键数据流图

### 6.1 Agent 执行主流程

```
用户消息到达
  ↓
路由 → 确定 Session Key → 加载 SessionEntry
  ↓
runEmbeddedPiAgent()
  ↓
┌─ while loop ──────────────────────────┐
│  1. 组装工具集（7 层过滤）            │
│  2. 构建系统提示词                     │
│  3. 加载会话历史                       │
│  4. streamSimple() → LLM              │
│  5. 处理响应（文本 / 工具调用）        │
│  6. 检查 compaction                    │
│     ├── 需要 → Memory Flush → 压缩    │
│     └── 不需要 → 继续或退出           │
└───────────────────────────────────────┘
  ↓
保存 SessionEntry → 写入 Transcript
```

### 6.2 记忆系统数据流

```
                     ┌─── MEMORY.md ──────┐
                     │    memory/*.md      │
Agent 写入 ─────────→│                     │
                     └────────┬────────────┘
                              │ chokidar 监控
                              ↓
                     listMemoryFiles() 扫描
                              │
                              ↓ hash 比较
                     ┌────────┴────────────┐
                     │ 变化的文件           │
                     │  → chunkMarkdown()  │
                     │  → generateEmbedding│
                     │  → 写入 SQLite      │
                     └────────┬────────────┘
                              │
              ┌───────────────┼───────────────┐
              ↓               ↓               ↓
         chunks 表       chunks_vec 表    chunks_fts 表
         (原文分块)      (向量索引)       (全文索引)
              │               │               │
              └───────────────┼───────────────┘
                              ↓
                     memory_search 工具
                     (0.7×向量 + 0.3×BM25)
                              ↓
                     Agent 获取相关记忆
```

### 6.3 Session 索引数据流

```
Agent 对话 → appendMessage()
  ↓
installSessionToolResultGuard 拦截
  ↓
emitSessionTranscriptUpdate() → 事件广播
  ↓
MemoryIndexManager 事件监听
  ↓
pendingFiles → 5s 防抖 → delta 阈值检查
  ↓
dirtyFiles → sync → 解析 JSONL
  ↓
只提取 user/assistant 的 text 内容
  ↓
分块 → embedding → SQLite
```

---

## 关键源码文件索引

| 文件 | 主要内容 |
|------|----------|
| `src/agents/pi-embedded-runner/run.ts` | Agent 主循环 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | 单次执行尝试 |
| `src/agents/pi-tools.ts` | 工具组装流水线 |
| `src/agents/pi-tools.schema.ts` | Schema 标准化 |
| `src/agents/openclaw-tools.ts` | OpenClaw 自有工具 |
| `src/agents/system-prompt.ts` | 系统提示词构建 |
| `src/agents/compaction.ts` | 上下文压缩算法 |
| `src/agents/workspace.ts` | 工作区 bootstrap |
| `src/agents/tools/memory-tool.ts` | 记忆搜索/读取工具 |
| `src/agents/tools/browser-tool.ts` | 浏览器自动化工具 |
| `src/config/sessions/store.ts` | Session Store 并发安全 |
| `src/config/sessions/types.ts` | SessionEntry 类型定义 |
| `src/config/sessions/transcript.ts` | JSONL 记录写入 |
| `src/memory/manager.ts` | 记忆索引管理器 |
| `src/memory/internal.ts` | 文件扫描、分块、哈希 |
| `src/memory/hybrid.ts` | 混合搜索算法 |
| `src/memory/memory-schema.ts` | SQLite 表结构 |
| `src/memory/sync-memory-files.ts` | 记忆文件同步 |
| `src/memory/session-files.ts` | Session 内容提取 |
| `src/sessions/transcript-events.ts` | 事件总线 |
| `src/auto-reply/reply/memory-flush.ts` | Memory Flush 设置 |
| `src/auto-reply/reply/agent-runner-memory.ts` | Memory Flush 执行 |
| `src/plugins/tools.ts` | 插件工具加载 |
| `src/agents/bash-tools.ts` | Bash 执行工具 |
