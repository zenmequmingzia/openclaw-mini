# OpenClaw Mini

**OpenClaw 核心架构的精简复现，用于学习 AI Agent 的系统级设计。**

> "没有记忆的 AI 只是函数映射，有记忆 + 主动唤醒的 AI，才是会演化的'生命系统'"

## 为什么做这个项目

网上大多数 Agent 教程只讲 Agent Loop：

```python
while tool_calls:
    response = llm.generate(messages)
    for tool in tools:
        result = tool.execute()
        messages.append(result)
```

**这不是真正的 Agent 架构。** 一个生产级 Agent 需要的是"系统级最佳实践"。

OpenClaw 是一个超 43w 行的复杂 Agent 系统，本项目从中提炼出核心设计与最小实现，帮助你理解：

- Agent Loop 的双层循环与 EventStream 事件流
- 会话持久化与上下文管理（裁剪 + 摘要压缩）
- 长期记忆、技能系统、主动唤醒的真实实现
- 多 Provider 适配（Anthropic / OpenAI / Google / Groq 等 22+ 提供商）

## 模块分层

本项目按学习价值分为三层，建议按 **核心 → 扩展 → 工程** 的顺序阅读：

```
┌─────────────────────────────────────────────────────────────┐
│                     [工程层] Production                      │
│  生产级防护与控制，学习可跳过                                 │
│                                                              │
│  session-key · tool-policy · command-queue                   │
│  sandbox-paths · context-window-guard · tool-result-guard    │
├─────────────────────────────────────────────────────────────┤
│                     [扩展层] Extended                         │
│  openclaw 特有的高级功能，非通用 Agent 必需                   │
│                                                              │
│  Memory (长期记忆) · Skills (技能系统) · Heartbeat (主动唤醒) │
├─────────────────────────────────────────────────────────────┤
│                      [核心层] Core                            │
│  任何 Agent 都需要的基础能力 ← 优先阅读                      │
│                                                              │
│  Agent Loop (双层循环)     EventStream (18 种类型化事件)      │
│  Session (JSONL 持久化)    Context (加载 + 裁剪 + 摘要压缩)  │
│  Tools (工具抽象+内置)     Provider (多模型适配)              │
└─────────────────────────────────────────────────────────────┘
```

### 核心层 — 必读

| 模块 | 文件 | 核心职责 | openclaw 对应 |
|------|------|----------|---------------|
| **Agent** | `agent.ts` | 入口 + subscribe/emit 事件分发 | `agent.js` |
| **Agent Loop** | `agent-loop.ts` | 双层循环 (outer=follow-up, inner=tools+steering) | `agent-loop.js` |
| **EventStream** | `agent-events.ts` | 18 种 MiniAgentEvent 判别联合 + 异步推拉 | `types.d.ts` AgentEvent |
| **Session** | `session.ts` | JSONL 持久化、历史管理 | `session-manager.ts` |
| **Context** | `context/loader.ts` | 按需加载 AGENTS.md 等 bootstrap 文件 | `bootstrap-files.ts` |
| **Pruning** | `context/pruning.ts` | 三层递进裁剪 (tool_result → assistant → 保留最近) | `context-pruning/pruner.ts` |
| **Compaction** | `context/compaction.ts` | 自适应分块摘要压缩 | `compaction.ts` |
| **Tools** | `tools/*.ts` | 工具抽象 + 7 个内置工具 | `src/tools/` |
| **Provider** | `provider/*.ts` | 多模型适配层 (基于 pi-ai, 22+ 提供商) | `pi-ai` |

### 扩展层 — 选读

| 模块 | 文件 | 核心职责 | openclaw 对应 |
|------|------|----------|---------------|
| **Memory** | `memory.ts` | 长期记忆 (关键词检索 + 时间衰减) | `memory/manager.ts` |
| **Skills** | `skills.ts` | SKILL.md frontmatter + 触发词匹配 | `agents/skills/` |
| **Heartbeat** | `heartbeat.ts` | 两层架构: wake 请求合并 + runner 调度 | `heartbeat-runner.ts` + `heartbeat-wake.ts` |

### 工程层 — 可跳过

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| **Session Key** | `session-key.ts` | 多 agent 会话键规范化 (`agent:id:session`) |
| **Tool Policy** | `tool-policy.ts` | 工具访问三级控制 (allow/deny/none) |
| **Command Queue** | `command-queue.ts` | 并发 lane 控制 (session 串行 + global 并行) |
| **Tool Result Guard** | `session-tool-result-guard.ts` | 自动补齐缺失的 tool_result |
| **Context Window Guard** | `context-window-guard.ts` | 上下文窗口溢出保护 |
| **Sandbox Paths** | `sandbox-paths.ts` | 路径安全检查 |

---

## 核心设计解析

### 1. Agent Loop — 双层循环 + EventStream

**问题**：简单 while 循环无法处理 follow-up、steering injection、上下文溢出等复杂场景。

**openclaw 方案**：双层循环 + EventStream 事件流

```typescript
// agent-loop.ts — 返回 EventStream，IIFE 推送事件
function runAgentLoop(params): EventStream<MiniAgentEvent, MiniAgentResult> {
  const stream = createMiniAgentStream();

  (async () => {
    // outer loop: follow-up 循环（处理 end_turn / tool_use 继续）
    while (outerTurn < maxOuterTurns) {
      // inner loop: 工具执行 + steering injection
      // stream.push({ type: "tool_execution_start", ... })
    }
    stream.end({ text, turns, toolCalls });
  })();

  return stream;  // 调用方 for-await 消费
}
```

**事件订阅**（对齐 pi-agent-core `Agent.subscribe`）：

```typescript
const agent = new Agent({ apiKey, provider: "anthropic" });

const unsubscribe = agent.subscribe((event) => {
  switch (event.type) {
    case "message_delta":  // 流式文本
      process.stdout.write(event.delta);
      break;
    case "tool_execution_start":  // 工具开始
      console.log(`[${event.toolName}]`, event.args);
      break;
    case "agent_error":  // 运行错误
      console.error(event.error);
      break;
  }
});

const result = await agent.run(sessionKey, "列出当前目录的文件");
unsubscribe();
```

### 2. Session Manager — JSONL 持久化

**问题**：Agent 重启后如何恢复对话上下文？

```typescript
// session.ts — 追加写入，每行一条消息
async append(sessionId: string, message: Message): Promise<void> {
  const filePath = this.getFilePath(sessionId);
  await fs.mkdir(path.dirname(filePath), { recursive: true });
  await fs.appendFile(filePath, JSON.stringify(message) + "\n");
}
```

### 3. Context — 加载 + 裁剪 + 摘要压缩

**问题**：上下文窗口有限，如何在不丢失关键信息的情况下控制大小？

三层递进策略：
1. **Pruning** — 裁剪旧的 tool_result（保留最近 N 条完整）
2. **Compaction** — 超过阈值后，旧消息压缩为"历史摘要"
3. **Bootstrap** — 按需加载 AGENTS.md 等配置文件（超长文件 head+tail 截断）

### 4. Memory — 长期记忆 (扩展层)

**问题**：如何让 Agent "记住"跨会话的信息？

```typescript
// memory.ts — 关键词匹配 + 时间衰减
async search(query: string, limit = 5): Promise<MemorySearchResult[]> {
  const queryTerms = query.toLowerCase().split(/\s+/);
  for (const entry of this.entries) {
    let score = 0;
    for (const term of queryTerms) {
      if (text.includes(term)) score += 1;
      if (entry.tags.some(t => t.includes(term))) score += 0.5;
    }
    const recencyBoost = Math.max(0, 1 - ageHours / (24 * 30));
    score += recencyBoost * 0.3;
  }
}
```

openclaw 用 SQLite-vec 做向量语义搜索 + BM25 关键词搜索，本项目简化为纯关键词。

### 5. Heartbeat — 主动唤醒 (扩展层)

**问题**：Agent 如何"主动"工作，而不只是被动响应？

两层架构：
- **HeartbeatWake**（请求合并层）：多来源触发 (interval/cron/exec/requested) → 250ms 合并窗口 → 双重缓冲
- **HeartbeatRunner**（调度层）：活跃时间检查 → HEARTBEAT.md 解析 → 空内容跳过 → 重复抑制

| 设计点 | 为什么这样做 |
|--------|-------------|
| setTimeout 而非 setInterval | 精确计算下次运行时间，避免漂移 |
| 250ms 合并窗口 | 防止多个事件同时触发 |
| 双重缓冲 | 运行中收到新请求不丢失 |
| 重复抑制 | 24h 内相同消息不重复发送 |

---

## 设计模式索引

| 模式 | 所在文件 | 说明 |
|------|----------|------|
| EventStream 异步推拉 | `agent-events.ts` | push/asyncIterator/end/result |
| Subscribe/Emit 观察者 | `agent.ts` | listeners Set + subscribe 返回 unsubscribe |
| 双层循环 | `agent-loop.ts` | outer (follow-up) + inner (tools+steering) |
| JSONL 追加日志 | `session.ts` | 每行一条消息，追加写入 |
| 三层递进裁剪 | `context/pruning.ts` | tool_result → assistant → 保留最近 |
| 自适应分块摘要 | `context/compaction.ts` | 按 token 分块，逐块摘要 |
| 双重缓冲调度 | `heartbeat.ts` | running + scheduled 状态机 |
| 三级编译策略 | `tool-policy.ts` | allow/deny/none → 过滤工具列表 |

---

## 快速开始

```bash
cd examples/openclaw-mini
pnpm install

# Anthropic (默认)
export ANTHROPIC_API_KEY=sk-xxx
pnpm dev

# OpenAI
pnpm dev -- --provider openai
# (需要 OPENAI_API_KEY 环境变量)

# Google
pnpm dev -- --provider google
# (需要 GEMINI_API_KEY 环境变量)

# 指定模型
pnpm dev -- --provider openai --model gpt-4o

# 指定 agentId
pnpm dev -- --agent my-agent
```

## 使用示例

```typescript
import { Agent } from "openclaw-mini";

const agent = new Agent({
  provider: "anthropic",        // 支持 22+ 提供商
  // apiKey 不传则自动从环境变量读取
  agentId: "main",
  workspaceDir: process.cwd(),
  enableMemory: true,
  enableContext: true,
  enableSkills: true,
  enableHeartbeat: false,
});

// 事件订阅
const unsubscribe = agent.subscribe((event) => {
  if (event.type === "message_delta") {
    process.stdout.write(event.delta);
  }
});

const result = await agent.run("session-1", "请列出当前目录的文件");
console.log(`${result.turns} 轮, ${result.toolCalls} 次工具调用`);

unsubscribe();
```

## 学习路径建议

1. **核心层优先**：`agent-loop.ts` → `agent.ts` → `agent-events.ts` → `session.ts` → `context/`
2. **理解事件流**：subscribe/emit 模式 + EventStream 异步推拉
3. **扩展层选读**：`memory.ts` → `skills.ts` → `heartbeat.ts`（按兴趣）
4. **对照 openclaw 源码**：验证简化版是否抓住了核心
5. **工程层跳过**：除非你在做生产级 Agent，否则不需要关注

## License

MIT
