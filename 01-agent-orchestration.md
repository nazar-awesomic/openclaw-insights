# Agent Orchestration: Background Task Lifecycle

Інсайти з кодової бази OpenClaw для побудови системи фонової оркестрації агентів.

---

## 1. Agentic Loop (Цикл виконання агента)

**Ключові файли:**
- `src/agents/pi-embedded-runner/run/attempt.ts` — основний цикл
- `src/agents/pi-embedded-subscribe.ts` — підписка на події
- `src/auto-reply/reply/agent-runner-execution.ts` — обгортка з fallback

### Як працює цикл

Агент працює за принципом **tool-call loop**: модель генерує відповідь, якщо в ній є tool call — система виконує інструмент, повертає результат моделі, і цикл повторюється до тих пір, поки модель не дасть фінальну відповідь без tool calls.

```
User Prompt
    ↓
streamSimple(tools + prompt)  ←──────────────┐
    ↓                                         │
Model Response                                │
    ├─ Has tool_call? ──YES──→ Execute tool   │
    │                         Return result ──┘
    └─ No tool_call? ──→ Final response → Done
```

### Підписка на події (Subscription Pattern)

```typescript
// src/agents/pi-embedded-subscribe.ts:31-150
// Стан підписки зберігає:
const state = {
  assistantTexts: [],    // накопичений текст AI
  toolMetas: [],         // виконані інструменти
  blockBuffer: null,     // буфер thinking-блоків <think>...</think>
  deltaBuffer: "",       // стрімінг-акумуляція
};

// Колбеки:
onPartialReply   // кожен новий шматок тексту (для стрімінгу)
onBlockReply     // завершений блок тексту
onToolResult     // результат виконання tool
```

**Ключовий інсайт:** Агент не "знає" коли зупинитись. Це вирішує модель — коли вона повертає відповідь без tool_use, цикл закінчується.

---

## 2. Queue та Session Management

### Queue для повідомлень

**Файл:** `src/auto-reply/reply/agent-runner.ts:162-198`

OpenClaw вирішує проблему "що робити коли агент вже працює, а прийшло нове повідомлення" через **queue modes**:

| Режим | Поведінка |
|-------|-----------|
| `interrupt` | Зупиняє поточний run, починає новий |
| `steer` | Вставляє нове повідомлення у поточний контекст (steering) |
| `followup` | Ставить у чергу, виконає після завершення поточного |
| `steer-backlog` | Комбінація steer + followup |
| `collect` | Збирає повідомлення, відправляє все разом |
| `queue` | Ставить в чергу (варіант followup) |

```typescript
// Steering — вставка повідомлення в поточний run:
if (shouldSteer && isStreaming) {
  const steered = queueEmbeddedPiMessage(sessionId, prompt);
  if (steered && !shouldFollowup) {
    return undefined; // Повідомлення вставлено, не треба новий run
  }
}

// Followup — черга:
if (isActive && shouldFollowup) {
  enqueueFollowupRun(queueKey, followupRun, resolvedQueue);
  return undefined; // Повідомлення в черзі
}
```

### Chat Run Registry (Gateway рівень)

**Файл:** `src/gateway/server-chat.ts:41-91`

```typescript
// Реєстр активних runs на рівні gateway:
createChatRunRegistry()  // Менеджер активних runs по session key
createChatRunState()     // Стан конкретного run (буфер стрімінгу, deltas)
```

Кожен session key має максимум один активний run. Нові запити або стають в чергу, або переривають поточний.

---

## 3. Followup Run — Автоматичне продовження

**Тип:** `src/auto-reply/reply/queue/types.ts:21-82`
**Конструювання:** `src/auto-reply/reply/get-reply-run.ts:356-406`

Система створює `FollowupRun` об'єкт, який містить повний контекст для виконання:

```typescript
const followupRun: FollowupRun = {
  prompt: queuedBody,                    // Текст повідомлення
  messageId: ctx.MessageSidFull,         // ID повідомлення
  summaryLine: baseBodyTrimmedRaw,       // Короткий опис
  enqueuedAt: Date.now(),                // Час додавання в чергу
  originatingChannel: ctx.Channel,       // Канал-джерело
  run: {
    agentId,
    agentDir,
    sessionId,
    sessionKey,
    messageProvider,
    thinkLevel,
    model,
    workspace,
    config,
    // ... 20+ полів контексту
  }
};
```

**Інсайт для твого проекту:** Зберігай повний snapshot контексту при постановці задачі в чергу. Це дозволяє відновити виконання навіть якщо конфігурація змінилась.

---

## 4. Model Fallback Chain

**Файл:** `src/agents/pi-embedded-runner/model.ts:153-225`

Якщо основна модель fails, система автоматично переходить до fallback:

```typescript
// agent-runner-execution.ts:148-468
const fallbackResult = await runWithModelFallback({
  provider: "anthropic",
  model: "claude-sonnet-4-20250514",
  fallbacksOverride: [
    { provider: "openai", model: "gpt-4o" },
    { provider: "anthropic", model: "claude-3-haiku" }
  ],
  run: async (provider, model) => {
    return runEmbeddedPiAgent({ provider, model, ...context });
  }
});
```

---

## 5. Error Recovery — Auto-відновлення

**Файл:** `src/auto-reply/reply/agent-runner-execution.ts:503-594`

Система автоматично відновлюється від типових помилок:

```typescript
// Класифікація помилок:
const isContextOverflow = isLikelyContextOverflowError(message);
const isCompactionFailure = isCompactionFailureError(message);
const isSessionCorruption = /function call turn comes immediately after/i.test(message);
const isRoleOrderingError = /incorrect role information|roles must alternate/i.test(message);
```

| Помилка | Відновлення |
|---------|-------------|
| Context overflow | Скидає сесію, починає з чистого контексту |
| Role ordering | Видаляє пошкоджений транскрипт, перезапускає |
| Session corruption | Видаляє пошкоджений запис з store |
| Model error | Переходить до fallback моделі |

**Інсайт для твого проекту:** Зроби кілька рівнів recovery:
1. Retry з тією ж моделлю
2. Fallback на іншу модель
3. Скидання контексту і повторна спроба
4. Повідомлення юзеру з пропозицією дій

---

## 6. AbortSignal — Graceful Cancellation

**Файли:**
- `src/infra/backoff.ts:16-28` — `sleepWithAbort()`
- `src/agents/pi-tool-definition-adapter.ts` — передача signal в tools

```typescript
// Кожен tool отримує AbortSignal:
tool.execute(toolCallId, params, onUpdate, ctx, signal)

// AbortSignal пробрасується через весь стек:
sleepWithAbort(ms: number, signal?: AbortSignal): Promise<void>

// Graceful shutdown з таймаутом:
// src/cli/gateway-cli/run-loop.ts:38-42
const forceExitTimer = setTimeout(() => process.exit(1), 5000);
forceExitTimer.unref(); // Не тримає event loop
```

---

## 7. Typing Controller (Індикація "агент працює")

**Файл:** `src/auto-reply/reply/typing.ts:56-134`

```typescript
// Sealing pattern — запобігає зависанню typing indicator:
let sealed = false;

const cleanup = () => {
  if (sealed) return;
  clearTimeout(typingTtlTimer);
  clearInterval(typingTimer);
  sealed = true; // Late callbacks не зможуть перезапустити typing
};

// Typing зупиняється коли ОБИДВІ умови виконані:
// 1. Agent run завершився (markRunComplete)
// 2. Черга dispatcher'а порожня (markDispatchIdle)
const maybeStopOnIdle = () => {
  if (runComplete && dispatchIdle) cleanup();
};
```

**Інсайт для твого проекту:** Індикатор "в процесі" повинен зупинятись тільки коли і генерація, і доставка результату завершені. Використовуй "sealing" щоб уникнути race conditions.

---

## 8. Session Write Lock (Критичний патерн)

**Файл:** `src/agents/pi-embedded-runner/run/attempt.ts:403-405, 922`

Запобігає конкурентним мутаціям сесії:

```typescript
// Перед початком agentic loop:
const releaseLock = await acquireSessionWriteLock(sessionId);
try {
  // ... весь agentic loop тут ...
} finally {
  releaseLock(); // Завжди звільняється
}
```

**Інсайт для твого проекту:** Якщо юзер може мати кілька активних задач, кожна задача повинна мати write lock на свій workspace. Запобігає корупції файлів при паралельній генерації.

---

## 9. Hook Runner Integration

**Файл:** `src/agents/pi-embedded-runner/run/attempt.ts:710-749, 854-873`

Плагіни можуть втручатись в lifecycle агента:

```typescript
// before_agent_start — плагіни можуть додати контекст до system prompt:
const hookResult = await hookRunner.run("before_agent_start", {
  agentId, sessionKey, prompt, systemPrompt
});
// hookResult може містити: { systemPrompt?, prependContext? }

// agent_end — fire-and-forget (не блокує відповідь):
void hookRunner.run("agent_end", { agentId, result, toolMetas });
```

**Інсайт:** `before_agent_start` — await (блокує), `agent_end` — fire-and-forget (не блокує). Обирай стратегію в залежності від критичності хука.

---

## 10. Session Compaction та History Repair

### Compaction Retry

**Файл:** `src/agents/pi-embedded-runner/run/attempt.ts:833-842`

Коли контекст стає занадто великим, система автоматично компактує (стискає) історію:

```typescript
// Якщо compaction fails — автоматичний retry:
await waitForCompactionRetry();
// Graceful degradation без втрати conversation state
```

### History Repair

**Файл:** `src/agents/pi-embedded-runner/run/attempt.ts:757-771`

Автоматичне виявлення та виправлення "осиротілих" user messages:

```typescript
// Виявляє порушення порядку ролей (user-user без assistant між ними)
// Запобігає role ordering violations від session manipulation
// Критично для multi-agent scenarios
```

---

## 11. Reasoning Level Support

**Файли:** `src/agents/pi-embedded-subscribe.ts:45`, system-prompt.ts

Три режими reasoning (thinking):
- `"off"` — без reasoning
- `"on"` — reasoning є, але прихований від стріму
- `"stream"` — reasoning стрімиться окремим потоком

```typescript
// Reasoning — це окремий стрім, може буферизуватись інакше ніж основна відповідь
state.streamReasoning = true; // Прапорець для reasoning mode
```

---

## 12. Cron/Scheduling (Заплановані задачі)

OpenClaw підтримує cron-based задачі через gateway:

- Cron service ініціалізується при старті gateway (`src/gateway/server.impl.ts`)
- Agent tool `createCronTool()` дозволяє агентам створювати заплановані задачі
- Кожна задача прив'язана до agent + session

---

## 13. Execution Approval (Схвалення виконання)

**Файл:** `src/gateway/server-methods.ts:93-120`

Для небезпечних операцій (bash, file write) система використовує approval flow:

```
Agent requests tool execution
    ↓
ExecApprovalManager checks policy
    ├─ Auto-approved (read-only tools)
    ├─ Requires approval → send to UI/CLI → wait for user response
    └─ Denied by policy
```

Три рівні авторизації:
- `node` — тільки вузол (внутрішні операції)
- `operator` — оператор з рівнями `admin`, `read`, `write`
- `public` — публічні методи

---

## 14. Патерн для твого проекту: Task Lifecycle

На основі OpenClaw, рекомендована архітектура для "landing page builder":

```
┌─────────────────────────────────────────────────┐
│                  USER REQUEST                    │
│  "Створи landing page для фітнес-студії"        │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│            TASK QUEUE (Redis/DB)                 │
│  status: pending → in_progress → review →       │
│          fixing → review → completed             │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│          AGENT ORCHESTRATOR                      │
│  1. Pick task from queue                         │
│  2. Create session context                       │
│  3. Run agentic loop:                            │
│     - Generate code (tool: write_file)           │
│     - Preview (tool: screenshot)                 │
│     - Verify (tool: check_design)                │
│     - Fix issues (tool: edit_file)               │
│     - Re-verify                                  │
│  4. Update task status                           │
│  5. Notify user                                  │
└─────────────────────────────────────────────────┘
```

### Ключові компоненти:

1. **Task Queue** — Redis або PostgreSQL з polling
2. **Agent Runner** — процес який забирає задачі з черги
3. **Session Store** — JSONL файл або DB для історії conversation
4. **Tool Registry** — набір tools для генерації, перевірки, фіксів
5. **Status Broadcaster** — WebSocket для real-time оновлень юзеру
6. **Retry/Recovery** — автоматичне відновлення при помилках

### Agentic Loop для Landing Page:

```typescript
async function executeTask(task: Task) {
  updateStatus(task.id, "in_progress");

  const tools = [
    writeFileTool,      // Записати HTML/CSS/JS
    screenshotTool,     // Зробити скріншот preview
    validateTool,       // Перевірити accessibility, responsive
    editFileTool,       // Виправити проблеми
    deployPreviewTool,  // Задеплоїти preview
  ];

  // Agentic loop — модель сама вирішує коли закінчити
  const result = await runAgent({
    prompt: task.userPrompt,
    systemPrompt: LANDING_PAGE_SYSTEM_PROMPT,
    tools,
    maxIterations: 20,           // Safety limit
    onToolCall: (tool, args) => {
      broadcastToUser(task.userId, { type: "tool_call", tool, args });
    },
    onPartialReply: (text) => {
      broadcastToUser(task.userId, { type: "streaming", text });
    },
  });

  updateStatus(task.id, "completed", { previewUrl: result.previewUrl });
  notifyUser(task.userId, "Landing page ready!");
}
```
