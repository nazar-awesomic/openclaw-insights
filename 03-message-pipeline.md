# Message Pipeline: від запиту юзера до результату

Інсайти з OpenClaw для побудови pipeline обробки запитів.

---

## 1. Повний Flow (Overview)

```
User Request (HTTP/WebSocket/API)
    ↓
Channel Listener (receive raw event)
    ↓
Message Handler (preflight, debounce, build context)
    ↓
Route Resolution (determine target agent)
    ↓
dispatchInboundMessage (main entry)
    ↓
getReplyFromConfig (session + directives + inline actions)
    ↓
runPreparedReply (queue mode, context assembly)
    ↓
runReplyAgent (queue steering, memory flush)
    ↓
runAgentTurnWithFallback (model fallback, error recovery)
    ↓
Embedded Pi Agent (agentic tool loop)
    ↓
ReplyDispatcher (serialize: tool → block → final)
    ↓
Channel Outbound (format + send to user)
```

---

## 2. Message Context (Уніфікований контекст)

**Файл:** `src/auto-reply/templating.ts`

Всі канали (Telegram, Discord, Web, etc.) конвертуються в єдиний `MsgContext`:

```typescript
type MsgContext = {
  Body: string;              // Текст повідомлення
  RawBody: string;           // Оригінальний текст
  From: string;              // Відправник
  To: string;                // Отримувач
  Surface: string;           // Канал (telegram, discord, web)
  Provider: string;          // Провайдер
  ChatType: "direct" | "group" | "channel";
  MediaTypes: string[];      // Типи вкладень
  MediaPath: string;         // Шлях до медіа
  SessionKey: string;        // Ключ сесії
  CommandSource: string;     // Джерело команди
  GroupId: string;           // ID групи
  GroupName: string;         // Назва групи
  MessageSid: string;        // ID повідомлення
  ThreadId: string;          // ID треду
  ReplyToId: string;         // ID повідомлення-відповіді
  AccountId: string;         // ID акаунту
};
```

**Інсайт для твого проекту:** Створи уніфікований `TaskContext`:

```typescript
type TaskContext = {
  taskId: string;
  userId: string;
  prompt: string;
  preferences: UserPreferences;
  previousIterations: Iteration[];
  attachments: Attachment[];
  targetPlatform: "web" | "mobile";
  style: StylePreferences;
};
```

---

## 3. Route Resolution (Маршрутизація)

**Файл:** `src/routing/resolve-route.ts:171-264`

OpenClaw вирішує який агент обробляє повідомлення через **bindings** (пріоритет):

1. Peer-specific binding (конкретний юзер → конкретний агент)
2. Parent peer binding (наслідування від треду)
3. Guild binding (Discord server)
4. Team binding (MS Teams)
5. Account binding (конкретний акаунт на каналі)
6. Channel-wide binding (весь канал)
7. Default agent (fallback)

**Session Key** — унікальний ключ `agent:channel:accountId:peer`:

```typescript
// src/routing/session-key.ts
// Забезпечує ізольовані conversation histories
// dmScope: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer"
```

---

## 4. Dispatch — Головна точка входу

**Файл:** `src/auto-reply/dispatch.ts:17-32`

```typescript
export async function dispatchInboundMessage(params: {
  ctx: MsgContext;
  cfg: Config;
  dispatcher: ReplyDispatcher;
  replyOptions?: GetReplyOptions;
}): Promise<DispatchInboundResult>
```

Три варіанти dispatch:
- `dispatchInboundMessage` — прямий dispatcher
- `dispatchInboundMessageWithDispatcher` — створює dispatcher, чекає idle
- `dispatchInboundMessageWithBufferedDispatcher` — typing-aware з `markDispatchIdle()`

---

## 5. Reply Generation Pipeline

**Файл:** `src/auto-reply/reply/get-reply.ts:53-335`

### 7 стадій генерації відповіді:

**Stage 1: Session Resolution (60-101)**
```typescript
const agentId = resolveAgentIdFromSessionKey(sessionKey);
const skillFilter = mergeSkillFilters(channelFilter, agentFilter);
const workspaceDir = resolveWorkspaceDir(agentDir);
```

**Stage 2: Context & Media Understanding (117-130)**
```typescript
const finalized = finalizeInboundContext(ctx);
await applyMediaUnderstanding(finalized);  // Vision для зображень
await applyLinkUnderstanding(finalized);   // URL enrichment
```

**Stage 3: Session State (138-160)**
```typescript
const { sessionCtx, entry, store, resetTriggers } =
  await initSessionState({ agentId, sessionKey, config });
```

**Stage 4: Directives (177-205)**
```typescript
const directives = await resolveReplyDirectives({
  ctx, cfg, agentId, provider, model, ...
});
// Парсить: /think, /verbose, model selection, etc.
```

**Stage 5: Inline Actions (238-280)**
```typescript
const inlineResult = await handleInlineActions({
  command, skillCommands, directives, cleanedBody, ...
});
// Обробляє: !echo, !bash, etc.
```

**Stage 6: Media Staging (282-288)**
```typescript
await stageSandboxMedia(finalized, workspaceDir);
```

**Stage 7: Agent Execution (290-334)**
```typescript
return runPreparedReply({ commandBody, followupRun, ... });
```

---

## 6. Reply Dispatcher (Серіалізація відповідей)

**Файл:** `src/auto-reply/reply/reply-dispatcher.ts:101-164`

**Ключовий патерн — Promise Chain серіалізація:**

```typescript
function createReplyDispatcher(options): ReplyDispatcher {
  let sendChain: Promise<void> = Promise.resolve();
  let pending = 0;
  let sentFirstBlock = false;

  const enqueue = (kind: "tool" | "block" | "final", payload) => {
    pending++;
    const shouldDelay = kind === "block" && sentFirstBlock;
    if (kind === "block") sentFirstBlock = true;

    sendChain = sendChain
      .then(async () => {
        if (shouldDelay) {
          await sleep(getHumanDelay(options.humanDelay)); // 800-2500ms
        }
        await options.deliver(payload, { kind });
      })
      .catch(err => options.onError?.(err, { kind }))
      .finally(() => {
        pending--;
        if (pending === 0) options.onIdle?.();
      });
  };

  return {
    sendToolResult: (p) => enqueue("tool", p),
    sendBlockReply: (p) => enqueue("block", p),
    sendFinalReply: (p) => enqueue("final", p),
    waitForIdle: () => sendChain,
  };
}
```

**Гарантії:**
- Tool results завжди перед blocks
- Blocks завжди перед final
- Human-like затримка між blocks (800-2500ms)
- `onIdle()` тільки коли черга повністю порожня

---

## 7. Block Reply Pipeline (Стрімінг)

**Файл:** `src/auto-reply/reply/block-reply-pipeline.ts:72-242`

```typescript
function createBlockReplyPipeline(params) {
  let sendChain = Promise.resolve();
  let aborted = false;

  const sendPayload = (payload) => {
    if (aborted) return;

    const key = createBlockReplyPayloadKey(payload); // Дедуплікація
    if (seenKeys.has(key)) return;

    sendChain = sendChain
      .then(() => withTimeout(onBlockReply(payload), timeoutMs))
      .catch(err => {
        if (isTimeout(err)) {
          aborted = true; // Poison pill — зупиняє всі наступні блоки
        }
      });
  };

  const enqueue = (payload) => {
    if (hasMedia(payload)) {
      coalescer?.flush({ force: true }); // Текст перед медіа
      sendPayload(payload);
    } else if (coalescer) {
      coalescer.enqueue(payload); // Текст батчиться
    } else {
      sendPayload(payload);
    }
  };
}
```

**Ключові патерни:**
- **Coalescing** — текстові блоки об'єднуються перед відправкою
- **Media flush** — медіа завжди відправляється окремо
- **Poison pill** — при timeout весь pipeline зупиняється
- **Deduplication** — payload key запобігає дублюванню

---

## 8. Debouncing (Групування швидких повідомлень)

**Файл:** `src/auto-reply/inbound-debounce.ts`

```typescript
// Групує швидкі повідомлення (default: 100ms)
// Запобігає:
// - API spam
// - Redundant processing
// - Out-of-order replies
```

**Файл:** `src/discord/monitor/message-handler.ts:44-137`

Discord-specific debouncing для групування rapid messages.

---

## 9. Cross-Channel Routing

**Файл:** `src/auto-reply/reply/dispatch-from-config.ts:200-212`

```typescript
// Якщо повідомлення прийшло з одного каналу,
// але відповідь потрібно відправити в інший:
const shouldRouteToOriginating =
  isRoutableChannel(originatingChannel) &&
  originatingTo &&
  originatingChannel !== currentSurface;
```

---

## 10. Патерн для твого проекту: Request Pipeline

```typescript
// task-pipeline.ts
async function processTaskRequest(request: TaskRequest) {
  // 1. Валідація і нормалізація
  const context = normalizeTaskContext(request);

  // 2. Queue mode check
  const existingRun = getActiveRun(context.taskId);
  if (existingRun) {
    return handleExistingRun(existingRun, context);
  }

  // 3. Dispatch to agent
  const dispatcher = createResultDispatcher({
    deliver: (result) => broadcastToUser(context.userId, result),
    onIdle: () => updateTaskStatus(context.taskId, "idle"),
    humanDelay: { minMs: 500, maxMs: 1500 },
  });

  // 4. Run agent with streaming
  const result = await runPageGenerator({
    context,
    onToolResult: (payload) => dispatcher.sendToolResult(payload),
    onBlockReply: (payload) => dispatcher.sendBlockReply(payload),
  });

  // 5. Send final result
  dispatcher.sendFinalReply(result);
  await dispatcher.waitForIdle();

  return { taskId: context.taskId, status: "completed" };
}
```

### Streaming для Landing Page Builder:

```typescript
// Юзер бачить прогрес в реальному часі:
onToolResult: (payload) => {
  // "Generating HTML structure..."
  // "Adding Tailwind classes..."
  // "Creating responsive layout..."
  ws.send({ type: "tool_progress", ...payload });
}

onBlockReply: (payload) => {
  // Стрімінг HTML preview
  ws.send({ type: "preview_update", html: payload.text });
}
```
