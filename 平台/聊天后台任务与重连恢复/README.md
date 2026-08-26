# 流式聊天后台化、队列化与 UI/LLM 解耦

## 1. 目标

聊天链路按“UI 是观察者，LLM turn 是后台任务”设计。

核心目标：

- 用户输入先进入 Redis 输入队列，由后台 worker 消费后执行 Agent。
- `worker-count` 控制当前 JVM 同时执行的完整聊天 turn 数量。
- worker 取到任务后一直等到本轮流式回答终态，才继续消费下一条任务。
- 超过 `worker-count` 的输入继续留在 Redis 队列中等待；单会话排队上限由 `max-pending-per-session` 控制（默认 3，设为 0 表示不限制）。
- 页面刷新、切换路由、退出登录、网络断开只断开 UI 连接，不中断后台 Agent 任务。
- 运行中的回答用 Redis snapshot 保存“已生成全文 + 轻量状态轨迹”，用于页面重新点进后恢复。
- 输出不使用 Redis 阻塞队列抢消费，而是 Redis Pub/Sub 广播到所有节点，再由有 WebSocket 的节点推送给本机客户端。
- 用户主动停止走独立 turn 控制面，停止 queued / running / 委派子 Agent。

设计约束：

- 不要 output 跨节点阻塞消费；输出走 Redis Pub/Sub 广播。
- worker 默认使用 JDK 虚拟线程。

当前实现仍保留 `chat-queue.enabled`、`max-pending-per-session` 等配置项，生产环境建议保持默认开启。

## 2. 核心标识

| 标识 | 语义 | 用途 |
|------|------|------|
| `contextId` | 对话会话 ID | 区分历史会话 |
| `agentId` | 当前会话 Agent ID | 与 `contextId` 组成会话执行维度 |
| `subscriptionId` | WebSocket 连接 ID | 区分同一会话的不同页面、不同客户端 |
| `turnId` | 后台真实执行轮次 ID | snapshot、取消、终态合并的权威 ID |
| `contextId + agentId` | session key | 输入串行、输出广播、主动停止的维度 |

注意：

- `subscriptionId` 不是任务维度。
- 同一用户多个客户端打开同一个 `contextId + agentId` 时，是共同观察同一个后台 turn。
- 主 Agent 委派子 Agent 时，UI 仍然按父 `turnId` 展示一轮回答。

## 3. 总体流程

```mermaid
flowchart TD
  UI["前端 UI"] --> WS["ChatController WebSocket"]

  WS --> ENQ["ChatInputQueueManager.enqueue"]
  ENQ --> SQ["Redis session input queue<br/>{app}:chat:input:{contextId}:{agentId}"]
  ENQ --> RQ["Redis ready queue<br/>{app}:chat:input:ready"]

  RQ --> WK["ChatInputQueueWorker<br/>固定数量虚拟线程"]
  WK --> TAKE["取出一个 ChatTurnInputTask"]
  TAKE --> LOCK["Redis 会话锁<br/>contextId + agentId"]
  LOCK --> CS["ChatService.handleChat"]

  CS --> TC["ChatTurnControlService<br/>注册真实 turnId"]
  CS --> LLM["主 Agent / 委派子 Agent"]
  LLM --> SNAP["ChatOutputEventCache<br/>全文 snapshot + stateTrail"]
  LLM --> OUT["ChatOutputDispatcher"]
  OUT --> PUB["Redis Pub/Sub output topic"]
  PUB --> NODE["各节点本机 callback registry"]
  NODE --> SEND["WebSocketSession.sendMessage"]
  SEND --> UI

  UIX["刷新 / 断网 / 路由切走 / 退出登录"] --> CLOSE["只移除当前 callback"]
  CLOSE --> KEEP["后台 turn 继续运行"]
  KEEP --> SNAP

  OPEN["用户点进运行中会话"] --> RESUME["resume=true WebSocket"]
  RESUME --> READ["读取 Redis snapshot"]
  READ --> REPLAY["补状态轨迹 + 全文快照"]
  REPLAY --> SEND

  STOP["用户主动停止"] --> API["Stop API"]
  API --> TC
  TC --> CANCEL["写 turn cancelled key<br/>发布 cancel topic"]
  CANCEL --> CUT["运行节点 dispose 主流 / 子流"]
  API --> OUT
```

## 4. Input 派

Input 派处理“UI 输入进入后台执行”的链路。

| 类 | 职责 |
|----|------|
| `ChatTurnInputTask` | 一轮聊天输入任务 DTO |
| `ChatInputQueueManager` | 生成 Redis key、写入 session queue、投递 ready 标记 |
| `ChatInputQueueWorker` | 固定数量虚拟线程阻塞消费 Redis 队列，调用 `ChatService` |
| `ChatQueueProperties` | 配置 worker 数和 Redis TTL |

### 4.1 ChatTurnInputTask

任务字段固定：

```java
class ChatTurnInputTask {
    String contextId;
    String agentId;
    String subscriptionId;
    String turnId;
    ChatRequestDto request;
    UserContextBo userContext;
    long enqueueTimeMs;
}
```

### 4.2 Redis 队列

使用两级队列：

```text
{app}:chat:input:{contextId}:{agentId}
{app}:chat:input:ready
```

- session input queue 保存真实 `ChatTurnInputTask`。
- ready queue 只保存 `contextId + agentId`，用于唤醒 worker。
- worker 先从 ready queue 拿到 session key，再从对应 session input queue 取任务。
- 任务执行完成后，如果该 session input queue 还有任务，再投递一次 ready 标记。

这样可以同时满足：

- 同一 `contextId + agentId` 串行。
- 不需要 worker 扫描大量 Redis key。
- 不同 session 可以被不同 worker 并发消费。

### 4.3 worker-count 语义

`worker-count` 是当前 JVM 的完整 turn 并发上限。

```yaml
chat-queue:
  worker-count: 8
```

含义：

- 当前 JVM 启动 8 个虚拟线程 worker。
- 每个 worker 一次只处理一个 `ChatTurnInputTask`。
- worker 进入 `ChatService.handleChat` 后，要等到本轮流式回答 `COMPLETED` / `FAILED` / `CANCELLED`，才继续取下一条任务。
- 所以当前 JVM 最多同时 running 8 个完整聊天 turn。
- 第 9 个及之后的输入不会失败，继续留在 Redis 队列里排队。

`worker-count: 1` 的含义：

- 当前 JVM 只有 1 个 worker 消费 input 队列。
- 只允许 1 个完整聊天 turn 处于 running。
- 其他输入都应该停留在 Redis 队列，等待前一个 turn 结束。

多 JVM 注意：

- `worker-count` 是单 JVM 配置。
- 如果部署 3 个 JVM，且每个 JVM `worker-count: 8`，集群理论入口并发是 24。
- 如果本地调试 `worker-count: 1` 仍看到多个独立会话同时输出，优先检查是否启动了多个后端进程，或是否存在绕过队列直接调用 `ChatService` 的入口。

### 4.4 pending 语义

- pending：还留在 Redis input queue 中、尚未被 worker 取出执行的输入任务。
- running：已经被 worker 取出、进入 `ChatService.handleChat`、正在等待流式终态的后台 turn。
- pending 可以积压；单会话上限由 `max-pending-per-session` 控制（默认 3）。
- running 由 `worker-count` 控制。

## 5. ChatService 执行

`ChatService` 仍然是业务核心，不变成队列组件。

主要职责：

- 路由真实 Agent。
- 生成真实 `turnId`。
- 注册 active turn。
- 注册 turn 控制面。
- 维护 Agent 状态机。
- 处理工具调用、委派子 Agent、记忆写入。
- 累加 `answerContent` / `reasoningContent`。
- 刷新 Redis snapshot。
- 输出事件交给 `ChatOutputDispatcher`。

执行流程：

```mermaid
sequenceDiagram
  participant W as ChatInputQueueWorker
  participant CS as ChatService
  participant TC as ChatTurnControlService
  participant AG as Agent
  participant CACHE as ChatOutputEventCache
  participant OUT as ChatOutputDispatcher

  W->>CS: handleChat(task)
  CS->>CS: resolve agentId / create turnId
  CS->>TC: registerTurn(contextId, agentId, turnId)
  CS->>OUT: dispatch THINKING / START
  CS->>AG: subscribe stream
  loop token delta
    AG-->>CS: answerDelta / reasoningDelta
    CS->>CS: append 到内存全文
    CS->>CACHE: saveSnapshot(全文)
    CS->>OUT: dispatch delta event
  end
  CS->>OUT: dispatch terminal event
  CS->>CACHE: clearSnapshot
  CS->>TC: unregisterTurn
  CS-->>W: terminal callback
```

关键点：

- queued 阶段不算 running。
- 只有进入 `ChatService` 后才有真实 `turnId` 和 active turn。
- worker 必须等 terminal callback 后才能释放该 turn 的执行槽。
- 回合开始即预落库 user 消息（所有 Agent，含非委派），避免工具等待期间刷新后历史里连提问都没有。
- 事件流上会按节流间隔 `touch` active-chat-turn 心跳；工具等待期间若仍有 TOOL/SYSTEM 等事件，心跳可续期。snapshot 主要靠 `saveStateTrailEvent` / `saveSnapshot` 写入时顺带续 TTL。
- 终态时才 `flushTurnTrace` 写入记忆中的 `turn_trace` 审计行；工具等待中途历史接口看不到完整 `turnSteps`。

记忆写入与“刷新后历史能看到什么”（工具等待中途）：

| 模式 | 历史里可见 | 历史里隐藏 / 尚无 |
|------|------------|-------------------|
| 非委派直聊 | user（预落库） | `assistant_tool_calls`（`displayInChat=false`）；尚无最终正文、`turn_trace` |
| 委派（子 Agent 执行中） | user（预落库，写在 `universal_assistant` 会话键） | 子 Agent 不写父会话记忆；尚无最终正文、`turn_trace` |

## 6. Output 派

Output 派处理“Agent 输出如何到达所有正在观察的 UI”。

| 类 | 职责 |
|----|------|
| `ChatCallbackRegistry` | 本机保存真实 WebSocket callback；Redis 保存 subscription 元数据和取消标记 |
| `ChatOutputDispatcher` | 发布 output 事件、广播 session 事件、串行写 WebSocket |
| `ChatOutputEventCache` | 保存运行中全文 snapshot 和轻量状态轨迹 |
| `ChatOutputSnapshot` | snapshot DTO |

### 6.1 为什么 output 不做阻塞队列

WebSocket session 只存在于当前 JVM。如果 output 做 Redis blocking queue，会出现：

- 事件可能被没有目标 WebSocket 的节点消费。
- 多客户端观察同一会话时，单消费者模型会导致其他客户端收不到。
- 取消、重连、终态合并会变复杂。
- output 顺序应该由单个 turn 内部状态机保证，不应该靠跨节点抢消费保证。

所以 output 用广播模型：

```mermaid
flowchart LR
  EVT["Agent event"] --> DISP["ChatOutputDispatcher"]
  DISP --> TOPIC["Redis Pub/Sub output topic"]
  TOPIC --> A["节点 A"]
  TOPIC --> B["节点 B"]
  TOPIC --> C["节点 C"]
  A --> A1["本机客户端 1"]
  A --> A2["本机客户端 2"]
  B --> B1["本机客户端 3"]
```

### 6.2 连接私有事件与 session 广播事件

连接私有事件：

- WebSocket connected。
- resume-empty。
- 参数错误、握手失败。

这些只发给当前 `subscriptionId`。

session 广播事件：

- Agent 状态事件。
- answer delta。
- reasoning delta。
- tool / orchestration 状态。
- COMPLETED / FAILED / CANCELLED。

这些按 `contextId + agentId` 广播给所有正在观察的客户端。

## 7. Callback Registry

`ChatCallback` 持有真实 `WebSocketSession`，不能放 Redis，也不能跨 JVM 直接调用。

因此边界是：

- 本机内存保存真实 callback。
- Redis 保存 subscription 元数据、session 订阅集合、取消标记。

```mermaid
flowchart TB
  subgraph JVM["当前 JVM"]
    MAP["callback map<br/>contextId:agentId:subscriptionId -> callback"]
    SESS["session map<br/>contextId:agentId -> subscriptionIds"]
  end

  subgraph REDIS["Redis"]
    CUR["callback current key"]
    SUB["subscription set"]
    SC["session cancelled key"]
    CC["subscription cancelled key"]
  end

  MAP -.注册元数据.-> CUR
  SESS -.注册订阅集合.-> SUB
  SC --> WORKER["queued 执行前检查"]
  CC --> WORKER
```

参考 key：

```text
{app}:chat:callback:current:{contextId}:{agentId}
{app}:chat:callback:subscriptions:{contextId}:{agentId}
{app}:chat:callback:cancelled-session:{contextId}:{agentId}
{app}:chat:callback:cancelled:{contextId}:{agentId}:{subscriptionId}
```

## 8. Snapshot 重连恢复

Snapshot 只服务“运行中的回答恢复”，不是历史记录。

Redis key：

```text
{app}:chat:output:snapshot:{contextId}:{agentId}
```

内容：

| 字段 | 说明 |
|------|------|
| `contextId` | 会话 ID |
| `agentId` | Agent ID |
| `turnId` | 当前真实 turn |
| `answerContent` | 当前已生成回答全文（可为空：工具等待期往往还没有正文） |
| `reasoningContent` | 当前已生成思考全文 |
| `state` | 当前状态 |
| `updatedAt` | 更新时间 |
| `stateTrail` | 轻量状态轨迹（可回放的事件序列） |

写入规则：

- 每次主 Agent 输出 text delta 后，用累计全文 `saveSnapshot`。
- 每次委派子 Agent 输出 delta 后，经 `SubAgentStreamBridge` 刷新**父** turn 的 snapshot。
- 任意 Agent-UI 事件经 `recordingResponseCall` 时调用 `saveStateTrailEvent`；若尚无 snapshot，会先建一条 `answerContent=""` 的骨架，再追加 trail——因此**工具等待期间也可能已有 snapshot**（只有步骤、没有正文），这是正常现象。
- snapshot 没有开关，默认就是后台任务恢复机制的一部分。
- TTL 只做异常兜底；有写入时会顺带续期。

`stateTrail` 的 payload 策略（resume 回放用）：

| 事件 | trail 中是否保留 payload | 原因 |
|------|--------------------------|------|
| `TOOL` / `SYSTEM` 等非 MESSAGE | 保留 | 恢复工具名、编排态等步骤 UI |
| `MESSAGE` + `phase=PATCH` | **保留** | 携带 `srcFile`、`pendingQuestion` 等补丁；清空会导致刷新后来源丢失 |
| `MESSAGE` + `phase=DELTA`（正文 token） | **置空** | 正文权威在 snapshot 全文；回放 delta 会重复 append |

后端 resume 下发顺序（`sendSnapshotIfPresent`）：

1. 按序回放 `stateTrail`（含 PATCH 上来源）。
2. 再发一条 `MESSAGE` + `snapshot=true` 的全文覆盖帧（`answerContent` / `reasoningContent`）。
3. 之后继续接实时 Pub/Sub 事件。

若无 snapshot，但 `activeChatTurnRegistry.isActive` 或输入队列仍有 pending，则只挂上观察连接、不发 `resume-empty`（等后续实时事件）。两者都没有才发 `notice: resume-empty`。

清理规则：

- turn `COMPLETED` / `FAILED` / `CANCELLED` 后清理 snapshot。
- 最终展示以历史记录接口为准。

## 9. 页面刷新与按需续传

前端不应该为了恢复运行中状态而自动连接**所有**会话的 WebSocket；但对“当前要展示的会话”会走 resume。

实际入口不止“侧边栏点历史”，还包括：

- 刷新后 `bootstrapAgentSession` 读 `ai.chat.browserLatestSession.v1`，自动 `showSessionView` 最近会话。
- 侧边栏点击历史会话。
- 全局任务浮窗点进某个运行中上下文。

正确模型：

```mermaid
flowchart TD
  BOOT["页面启动 / 点进会话"] --> LOCAL["读取 localStorage active turns"]
  LOCAL --> STORE["恢复 chatActivityStore 彩色球"]
  BOOT --> HIST["getHistoryContext 加载历史"]
  HIST --> RECON["reconcileRememberedTurnAfterHistoryLoad"]
  RECON --> DONE{"历史相对最新 user 已有可见终态?"}
  DONE -->|是| FIX["清理 running 标记<br/>不连接 resume WS"]
  DONE -->|否| REM{"localStorage / activity<br/>仍标记本会话活跃?"}
  REM -->|否| STATIC["只展示历史，不 resume"]
  REM -->|是| WS["建立 resume=true WebSocket"]
  WS --> SNAP{"后端 sendSnapshotIfPresent?"}
  SNAP -->|有| APPLY["回放 stateTrail<br/>再 snapshot 全文覆盖"]
  APPLY --> STREAM["继续接后续实时事件"]
  SNAP -->|无 active/队列| EMPTY["resume-empty → 清理标记"]
  SNAP -->|无 snapshot 但仍 active| WAIT["挂观察连接，等实时事件"]
```

### 9.1 “历史已终态”的真实判定

文档旧述“历史里已是 COMPLETED/FAILED/CANCELLED”不准确。前端用的是 `hasTerminalTurnAfterLatestUser`：

1. 找到**最后一条** `role=user`。
2. 只扫描它之后的消息。
3. **跳过** `displayInChat === false` 的行（含 `messageKind=TOOL_ROUND` 的 `assistant_tool_calls`、tool_result 等）。
4. 若存在**可见** assistant，且具备以下任一条件，则视为已终态并清理 running 标记、**不** resume：
   - 有正文 / 思考 / actions / srcFile；
   - `currentState` / `stateHistory` / `turnSteps` 含终态；
   - `turnSteps` 非空且全部非 `running`。

因此：工具等待中途历史上只有 `user + 隐藏 TOOL_ROUND` 时，**不会**被误判为终态，会继续 resume——这是预期行为。

### 9.2 前端锚点与可见气泡（易踩坑）

`messageContext` 保留完整历史（含隐藏行），渲染用 `visibleMessageContext` 过滤 `displayInChat === false`。

resume / 流式写入依赖 `ensureTurnAssistant` → `findReusableEmptyAssistantIndex`：在最新 user 之后找“正文为空的 assistant”复用。

**必须跳过不可渲染行**（`displayInChat !== false`）。否则会把历史里的隐藏 `assistant_tool_calls` 当成锚点：步骤条、正文、来源都写进隐藏行，页面上“提问在、助手气泡全无”。这是非委派工具等待期刷新后“续不上”的常见根因；委派模式因不落该隐藏行而天然免疫（见 §12）。

纠偏规则：

- 左侧彩色球代表“本地认为可能仍在运行”，不是绝对权威。
- 点进 / 自动重入会话后，先加载历史，再按 §9.1 纠偏。
- resume 收到 `resume-empty`：清理 localStorage 与 activity，不要把探测帧写入状态机。
- 工具等待期 snapshot 的 `answerContent` 可以为空：页面应至少能恢复步骤条；有 PATCH 时应恢复来源；正文等后续实时 delta。

## 10. UI 与 LLM 解耦

| 场景 | UI 行为 | 后端行为 |
|------|---------|----------|
| 页面刷新 | 断开当前 WebSocket，重新加载页面 | 只移除 callback，后台 turn 继续 |
| 切换路由 | 不弹“离开会中断任务”警告 | 后台 turn 继续 |
| 退出登录 | 只跳转 `/logout` | 不调用停止 |
| 网络断开 | 当前连接断开 | 后台 turn 继续 |
| 点进运行中会话 | 建立 `resume=true` 连接 | 返回 snapshot 并继续推流 |
| 用户主动停止 | 调 Stop API，本地 optimistic 收敛 | 取消 queued / running turn |

前端运行中状态来源：

- `localStorage ai.chat.activeTurns.v1`：刷新后恢复“哪些会话可能运行中”。
- `chatActivityStore`：当前页面运行中状态，驱动左侧彩色球、全局任务浮窗、输入框 busy 态。
- 历史记录和 resume 结果用于纠偏。

## 11. 主动停止控制面

停止不能依赖 WebSocket close。主动停止必须是稳定可寻址的 turn 控制面。

Stop API：

```text
POST /v1/rest/j2agent/chat/stop?context-id=...&agent-id=...
```

控制面 key：

```text
{app}:chat:turn:active:{contextId}:{agentId}
{app}:chat:turn:cancelled:{turnId}
{app}:chat:turn:cancel
```

流程：

```mermaid
sequenceDiagram
  participant UIA as 客户端 A
  participant API as Stop API
  participant CB as ChatCallbackRegistry
  participant TC as ChatTurnControlService
  participant R as Redis
  participant RUN as 运行节点
  participant OUT as ChatOutputDispatcher
  participant UIB as 客户端 B

  UIA->>API: stop(contextId, agentId)
  API->>CB: markSessionCancelled
  CB->>R: 写 session cancelled
  API->>TC: cancelSession(contextId, agentId)
  TC->>R: 读取 active turnId
  TC->>R: 写 turn cancelled key
  TC->>R: publish turn cancel
  R-->>RUN: 收到 cancel message
  RUN->>RUN: dispose 主 Agent stream
  RUN->>RUN: dispose 子 Agent stream
  RUN->>RUN: TurnCancellationGuard 阻止后续消费
  API->>OUT: cancelSession(realTurnId)
  OUT->>R: publish CANCELLED
  R-->>UIA: CANCELLED 同一 turnId
  R-->>UIB: CANCELLED 同一 turnId
```

要求：

- queued 任务：worker 执行前看到 session cancelled，直接丢弃。
- running 任务：运行节点 dispose Reactor stream。
- 主 Agent 和委派子 Agent 的 `Disposable` 都注册到同一个父 `turnId`。
- `CANCELLED` 事件必须使用真实 `turnId`，前端才能合并到当前 assistant 气泡。
- WebSocket close reason `user interrupt` 只做旧客户端兼容，不是权威停止路径。

## 12. 委派子 Agent

通用助手委派子 Agent 时，UI 仍然展示为父 turn 的一轮回答。

```mermaid
flowchart TD
  UA["UniversalAssistantOrchestratorService"] --> CALL["UniversalSubAgentCallService"]
  CALL --> SUB["子 Agent stream"]
  SUB --> BRIDGE["SubAgentStreamBridge"]
  BRIDGE --> TEXT["写回父 turn streamedContent / streamedReasoning"]
  BRIDGE --> SNAP["刷新父 turn snapshot"]
  BRIDGE --> OUT["按父 contextId + agentId 广播"]
  SUB --> DISP["注册子 Agent Disposable 到父 turnId"]
  STOP["停止父 session"] --> CANCEL["cancel 父 turnId"]
  CANCEL --> DISP
  DISP --> CUT["dispose 子 Agent stream"]
```

要求：

- 子 Agent 输出不能只从后半段开始恢复，必须写进父 snapshot 的全文。
- 子 Agent 状态必须进入父 snapshot 的 `stateTrail`。
- 子 Agent 取消检查必须走父 `turnId`。
- 父 session 停止时，子 Agent 不允许继续输出、继续落库、继续推送。
- 子 Agent 以 `subAgentCallRun=true` 运行：`ReactCompatibleMessageChatMemoryAdvisor` **跳过**对该子会话键的记忆读写，避免污染父会话历史。

### 12.1 为何委派模式下“工具等待期刷新”更容易看起来正常

重连协议对两种模式是同一套（snapshot + stateTrail + resume WS）。体感差异来自**数据是否写进父会话 / 父 snapshot**：

| 对比项 | 非委派直聊（工具执行中） | 委派（子 Agent / `sub_agent_call` 执行中） |
|--------|--------------------------|---------------------------------------------|
| 父会话记忆 | 已有隐藏 `assistant_tool_calls` | 通常只有预落库的 user；无隐藏空 assistant |
| 父 snapshot 正文 | 往往仍为空（本轮尚未产出可见文本） | 子 Agent 流式 token 持续写入父 `answerContent` |
| resume 后前端锚点 | 若误复用隐藏行 → 气泡不可见（见 §9.2） | 无隐藏空行 → 新建可见气泡 |
| 用户可见期望 | 至少：提问 + 步骤条（CALLING_TOOL）；正文可后到 | 提问 + 步骤条 + 往往已有部分正文 |

因此：**不要把“委派能续上、非委派不能”归因于重连机制分叉**；先查历史隐藏行是否污染锚点、PATCH 来源是否进了 trail、snapshot 是否仅有空正文。

## 13. 配置

当前配置只保留必要项：

```yaml
com:
  nms:
    ai:
      chat-queue:
        enabled: true                     # 默认启用 Redis 输入队列
        worker-count: 8                   # 当前 JVM 完整聊天 turn 并发上限
        max-pending-per-session: 3        # 单会话排队上限，0 表示不限制
        take-timeout-seconds: 5           # worker 从具体会话队列等待任务的超时时间
        queued-task-ttl-seconds: 300      # 空闲输入队列 key 兜底 TTL
        output-cache-ttl-seconds: 300     # 运行中 snapshot 兜底 TTL
      active-chat-turn:
        heartbeat-ttl-seconds: 30
        heartbeat-touch-interval-seconds: 10
        sweeper-interval-seconds: 60
        key-fallback-ttl-hours: 24
```

配置解释：

- `worker-count`：当前 JVM 同时执行多少个完整 turn。
- `max-pending-per-session`：同一 `contextId + agentId` 在 Redis 中最多排队的输入数；超出后向前端返回队列满失败事件。
- `take-timeout-seconds`：ready 标记可能重复，worker 从 session queue 等待任务的短超时。
- `queued-task-ttl-seconds`：输入队列 key 的异常兜底过期时间。
- `output-cache-ttl-seconds`：运行中 snapshot 兜底 TTL；snapshot 默认开启，回答完成后主动删除。

## 14. 状态流转

下图是**任务生命周期**示意（queued → running → 终态），不是 Agent 状态机全集。运行中还会出现 `CALLING_TOOL` / `LOAD_SKILL` / `AGENT_ORCHESTRATING` / `STREAMING_TEXT` 等，这些会进入 `stateTrail` 供 resume 回放。

```mermaid
stateDiagram-v2
  [*] --> QUEUED: UI 输入入队
  QUEUED --> RUNNING: worker 取出任务
  QUEUED --> CANCELLED: 用户停止 / session cancelled
  RUNNING --> THINKING: ChatService accepted
  THINKING --> STREAMING: Agent delta
  THINKING --> CANCELLED: 用户停止
  STREAMING --> COMPLETED: 正常完成
  STREAMING --> FAILED: 异常
  STREAMING --> CANCELLED: 用户停止
  COMPLETED --> [*]
  FAILED --> [*]
  CANCELLED --> [*]
```

前端展示：

- busy 态（含 `THINKING` / `CALLING_TOOL` / `LOAD_SKILL` / `STREAMING_TEXT` / `AGENT_ORCHESTRATING` 等）：显示运行中彩色球。
- `COMPLETED` / `FAILED` / `CANCELLED`：清理 activity 和 localStorage。
- 如果页面本地认为运行中，但历史相对最新 user 已有可见终态，或 resume 证明已空，立即纠偏。

## 15. 验证清单

- `worker-count: 1` 时，同一 JVM 不应同时执行两个独立聊天 turn。
- `worker-count: 8` 时，同一 JVM 最多 8 个完整 turn running，其余输入继续 pending。
- 同一 `contextId + agentId` 连续发送多条消息，应串行执行。
- 不同 `contextId + agentId` 在 worker 有空闲时可并发执行。
- 页面刷新、切换路由、退出登录不应中断后台任务。
- 左侧运行中彩色球不应自动为所有会话建立 WebSocket。
- 点进会话或刷新自动重入最近会话时，才对该会话建立 `resume=true` 连接。
- resume 后应先补状态轨迹（含 MESSAGE/PATCH 来源），再用 snapshot 全文覆盖 assistant 气泡，后续 delta 继续 append。
- 如果点进后发现历史相对最新 user 已有可见终态，或收到 `resume-empty`，应清理运行中标记。
- 非委派工具等待中刷新：应看到提问气泡 + 可见助手气泡上的步骤条；无正文时不应误判为“重连失败”。
- 同一用户多个客户端打开同一会话，应同时收到同一份输出。
- 任意客户端主动停止后，所有客户端都收到同一真实 `turnId` 的 `CANCELLED`。
- 委派子 Agent 运行中停止时，父流和子流都应被取消。

## 16. 排查重点

如果 `worker-count: 1` 仍看到多个独立会话同时输出，按顺序排查：

- 是否有多个后端 JVM 同时连接同一个 Redis。
- 是否有旧进程未停止。
- 是否有入口绕过 `ChatInputQueueManager.enqueue` 直接调用 `ChatService.handleChat`。
- worker 是否真的等到 terminal callback 后才释放。
- 委派子 Agent 是否被误认为独立 turn，而不是父 turn 内部步骤。

如果重连后只续到后半段，按顺序排查：

- 主 Agent delta 是否每次刷新 snapshot 全文。
- 委派子 Agent delta 是否写回父 turn snapshot。
- resume 是否读取 `contextId + agentId` 对应 snapshot。
- 前端 snapshot 事件是否覆盖 assistant 气泡，而不是 append。

如果工具等待期刷新后**提问在、助手气泡 / 步骤条全无**（协议层往往正常）：

1. 打开 resume WS 的 Messages：是否已有 `turn-started` / `TOOL START CALLING_TOOL` / `snapshot` 等回放帧。若有帧而页面无气泡 → 不是重连失败，是前端渲染锚点问题。
2. 查历史接口：最新 user 之后是否有 `displayInChat=false` 的空 assistant（`TOOL_ROUND`）。
3. 查前端 `findReusableEmptyAssistantIndex` / `isRenderableAssistant`：是否把隐藏行当成了本轮锚点。
4. 对照委派会话：父历史上不应有该隐藏行；若委派正常、非委派异常，优先查这一条。

如果刷新后**来源（srcFile）没了**，按顺序排查：

- `saveStateTrailEvent` 是否对 `MESSAGE` + `PATCH` **保留**了 payload（正文 `DELTA` 仍应置空）。
- resume 回放的 PATCH 帧 payload 里是否仍有 `message.srcFile`。
- 前端 `applyMessageDelta` 是否在收到 PATCH 时 `mergeSrcFiles`。

如果停止后 LLM 还在输出，按顺序排查：

- Stop API 是否查到真实 active `turnId`。
- turn cancelled key 是否写入 Redis。
- 运行节点是否收到 cancel topic。
- 主 Agent `Disposable` 是否注册到该 `turnId`。
- 子 Agent `Disposable` 是否注册到父 `turnId`。
- `TurnCancellationGuard` 是否覆盖主链路、工具链路、委派子链路。
