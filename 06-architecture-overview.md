# Architecture Overview: як це все поєднується

Загальний огляд архітектури OpenClaw та рекомендації для побудови landing page builder.

---

## 1. Високорівнева архітектура OpenClaw

```
┌──────────────────────────────────────────────────────────────┐
│                        GATEWAY                                │
│  (HTTP/WebSocket server, central orchestrator)                │
│                                                               │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌─────────────┐    │
│  │ Channel  │  │  Agent   │  │  Cron  │  │   Plugin    │    │
│  │ Manager  │  │ Events   │  │ Service│  │  Registry   │    │
│  └────┬─────┘  └────┬─────┘  └────┬───┘  └──────┬──────┘    │
│       │              │             │              │           │
│  ┌────▼──────────────▼─────────────▼──────────────▼────────┐ │
│  │              Gateway Method Handlers                     │ │
│  │  chat, channels, config, send, sessions, skills, models │ │
│  └──────────────────────┬──────────────────────────────────┘ │
└─────────────────────────┼────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
    │  Channel   │  │  Channel   │  │  Channel   │
    │  Plugins   │  │  Plugins   │  │  Plugins   │
    │ (Discord)  │  │ (Telegram) │  │  (Web)     │
    └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
          │               │               │
    ┌─────▼───────────────▼───────────────▼──────┐
    │           Auto-Reply System                 │
    │  dispatch → get-reply → agent-runner        │
    │  → embedded-pi-agent (agentic loop)         │
    └─────────────────────┬──────────────────────┘
                          │
    ┌─────────────────────▼──────────────────────┐
    │           Reply Dispatcher                  │
    │  tool → block → final (serialized)          │
    │  + coalescing + human delay + TTS           │
    └─────────────────────┬──────────────────────┘
                          │
    ┌─────────────────────▼──────────────────────┐
    │        Channel Outbound Adapters            │
    │  Format + chunk + send to user              │
    └────────────────────────────────────────────┘
```

---

## 2. Ключові архітектурні рішення OpenClaw

### 2.1 Single-Process Gateway
- Весь стейт в пам'яті одного процесу
- Координація через gateway lock (тільки один процес)
- Простота замість distributed архітектури

### 2.2 Plugin-First Architecture
- Все (канали, tools, hooks, CLI) — плагіни
- Core мінімальний, extension points скрізь
- Runtime loading через jiti (TypeScript без build step)

### 2.3 Session-per-Key Isolation
- Кожен `agent:channel:account:peer` має окрему сесію
- JSONL файли для persistence (не DB)
- Дозволяє паралельні conversations без конфліктів

### 2.4 Promise Chain Serialization
- Не використовує black-box queue libraries
- `sendChain = sendChain.then(...)` забезпечує порядок
- Простіше за message queue, достатньо для single-process

### 2.5 Model-Agnostic Tool Loop
- Tools описані через JSON Schema
- Модель (Claude, GPT, Gemini) взаємозамінна
- Schema normalization вирішує provider-specific quirks

---

## 3. Рекомендована архітектура для Landing Page Builder

```
┌─────────────────────────────────────────────────────────────┐
│                     WEB APPLICATION                          │
│  (Next.js / Remix / whatever)                                │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐     │
│  │  Dashboard    │  │  Task Queue  │  │  Preview      │     │
│  │  (React)      │  │  (API)       │  │  (iframe)     │     │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘     │
│         │                  │                   │             │
│  ┌──────▼──────────────────▼───────────────────▼───────────┐│
│  │              WebSocket Connection                        ││
│  │  (real-time progress, streaming, preview updates)        ││
│  └──────────────────────────┬──────────────────────────────┘│
└─────────────────────────────┼───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                      API SERVER                              │
│                                                              │
│  ┌────────────┐  ┌────────────────┐  ┌──────────────────┐  │
│  │  REST API   │  │  WebSocket     │  │  Auth + Billing  │  │
│  │  (tasks)    │  │  (streaming)   │  │  (Stripe)        │  │
│  └──────┬──────┘  └──────┬─────────┘  └──────────────────┘  │
│         │                │                                   │
│  ┌──────▼────────────────▼──────────────────────────────┐   │
│  │              Task Queue (Redis / PostgreSQL)          │   │
│  │  status: pending → in_progress → review → completed   │   │
│  └──────────────────────────┬───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                    WORKER PROCESS(ES)                         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Agent Orchestrator                       │    │
│  │                                                       │    │
│  │  1. Poll task from queue                              │    │
│  │  2. Create agent session                              │    │
│  │  3. Run agentic tool loop:                            │    │
│  │     ┌─────────────────────────────┐                   │    │
│  │     │  LLM (Claude/GPT)          │                   │    │
│  │     │  ↕ tool calls              │                   │    │
│  │     │  Tools:                     │                   │    │
│  │     │  - write_file               │                   │    │
│  │     │  - preview_screenshot       │                   │    │
│  │     │  - validate_page            │                   │    │
│  │     │  - edit_file                │                   │    │
│  │     │  - deploy_preview           │                   │    │
│  │     └─────────────────────────────┘                   │    │
│  │  4. Broadcast progress via WebSocket                  │    │
│  │  5. Update task status                                │    │
│  │  6. Handle errors + retry                             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Sandbox Environment                      │    │
│  │  - Isolated file system per task                      │    │
│  │  - Puppeteer for screenshots                          │    │
│  │  - Lighthouse for validation                          │    │
│  │  - Preview server (localhost:PORT)                    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Компоненти які потрібно реалізувати

### 4.1 Task Queue

```typescript
// Простий варіант — PostgreSQL:
interface TaskQueue {
  create(task: CreateTaskInput): Promise<Task>;
  claim(): Promise<Task | null>;  // Атомарний SELECT FOR UPDATE SKIP LOCKED
  update(id: string, update: TaskUpdate): Promise<void>;
  getByUser(userId: string): Promise<Task[]>;
  subscribe(taskId: string, ws: WebSocket): void;
}

// Redis варіант:
// BullMQ — production-ready queue на Redis
// Підтримує: priorities, retries, delayed jobs, rate limiting
```

### 4.2 Agent Orchestrator (ядро системи)

```typescript
class AgentOrchestrator {
  async executeTask(task: Task): Promise<TaskResult> {
    const session = await this.createSession(task);
    const tools = this.assembleTools(task);
    const systemPrompt = this.buildSystemPrompt(task);

    try {
      this.updateStatus(task.id, "in_progress");

      const result = await this.runAgentLoop({
        session,
        tools,
        systemPrompt,
        userPrompt: task.prompt,
        maxIterations: 30,
        timeoutMs: 5 * 60 * 1000, // 5 хвилин
        abortSignal: this.getAbortSignal(task.id),

        onToolCall: (tool, args, result) => {
          this.broadcast(task.id, { type: "tool_call", tool, args });
          this.broadcast(task.id, { type: "tool_result", result });
        },

        onStreaming: (text) => {
          this.broadcast(task.id, { type: "streaming", text });
        },
      });

      this.updateStatus(task.id, "completed", result);
      return result;

    } catch (err) {
      return this.handleError(task, err);
    }
  }

  private async runAgentLoop(params): Promise<AgentResult> {
    // Це серце системи — tool-call loop
    let messages = [
      { role: "system", content: params.systemPrompt },
      { role: "user", content: params.userPrompt },
    ];

    for (let i = 0; i < params.maxIterations; i++) {
      params.abortSignal.throwIfAborted();

      const response = await this.callModel(messages, params.tools);

      // Якщо немає tool calls — агент закінчив
      if (!response.toolCalls?.length) {
        return { text: response.text, iteration: i + 1 };
      }

      // Виконуємо tool calls
      const toolResults = await Promise.all(
        response.toolCalls.map(tc => this.executeTool(tc, params))
      );

      // Додаємо результати до контексту
      messages.push({ role: "assistant", content: response.raw });
      messages.push({ role: "tool", results: toolResults });
    }

    throw new Error("Max iterations reached");
  }
}
```

### 4.3 Result Dispatcher (broadcasting)

```typescript
// Адаптація ReplyDispatcher з OpenClaw:
class ResultDispatcher {
  private sendChain = Promise.resolve();
  private pending = 0;

  enqueue(event: ProgressEvent) {
    this.pending++;
    this.sendChain = this.sendChain
      .then(() => this.deliver(event))
      .catch(err => this.handleError(err))
      .finally(() => {
        this.pending--;
        if (this.pending === 0) this.onIdle?.();
      });
  }

  private deliver(event: ProgressEvent) {
    const subscribers = this.getSubscribers(event.taskId);
    for (const ws of subscribers) {
      try {
        ws.send(JSON.stringify(event));
      } catch {
        this.removeSubscriber(ws);
      }
    }
  }

  async waitForIdle() {
    await this.sendChain;
  }
}
```

### 4.4 Preview System

```typescript
// Кожна задача отримує ізольований preview:
class PreviewManager {
  async createPreview(taskId: string): Promise<PreviewInstance> {
    const port = await getAvailablePort();
    const dir = path.join(WORKSPACE_DIR, taskId);

    await fs.mkdir(dir, { recursive: true });

    // Простий static file server:
    const server = createStaticServer(dir);
    server.listen(port);

    return {
      url: `http://localhost:${port}`,
      dir,
      stop: () => server.close(),
    };
  }

  async screenshot(previewUrl: string, viewport: Viewport): Promise<Buffer> {
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    await page.setViewport(viewport);
    await page.goto(previewUrl, { waitUntil: "networkidle0" });
    const screenshot = await page.screenshot({ fullPage: true });
    await browser.close();
    return screenshot;
  }
}
```

---

## 5. Cycle: Generate → Verify → Fix → Re-verify

Це центральна ідея — agent loop з self-correction:

```
┌──────────────────────────────────────────────────────────┐
│                    SYSTEM PROMPT                          │
│                                                           │
│  "You are a landing page builder. Your workflow:          │
│   1. Analyze the user's request                          │
│   2. Generate HTML/CSS/JS using write_file               │
│   3. Take screenshot with preview_screenshot              │
│   4. Review the screenshot for issues                     │
│   5. If issues found → edit files → screenshot again      │
│   6. Run validate_page for accessibility & responsive     │
│   7. Fix validation issues → re-validate                  │
│   8. Deploy preview with deploy_preview                   │
│   9. Report completion with preview URL                   │
│                                                           │
│  Quality criteria:                                        │
│  - Responsive on desktop, tablet, mobile                  │
│  - Accessibility score > 90                               │
│  - All images have alt text                               │
│  - Color contrast meets WCAG AA                           │
│  - Page loads in under 3 seconds"                         │
│                                                           │
└──────────────────────────────────────────────────────────┘

Iteration 1: Generate
  → write_file("index.html", fullPage)
  → write_file("styles.css", styles)
  → preview_screenshot("desktop")
  → AI reviews: "Hero section text too small, CTA not prominent"

Iteration 2: Fix
  → edit_file("styles.css", fixFontSizes)
  → edit_file("index.html", improveCTA)
  → preview_screenshot("desktop")
  → preview_screenshot("mobile")
  → AI reviews: "Desktop looks good, mobile nav needs hamburger"

Iteration 3: Fix mobile
  → edit_file("index.html", addMobileNav)
  → edit_file("styles.css", addResponsive)
  → preview_screenshot("mobile")
  → validate_page(["accessibility", "responsive"])
  → Results: { score: 87, issues: ["missing alt on hero image"] }

Iteration 4: Fix accessibility
  → edit_file("index.html", addAltText)
  → validate_page(["accessibility"])
  → Results: { score: 95, issues: [] }

Iteration 5: Deploy
  → deploy_preview("/workspace/task-123")
  → "Your landing page is ready at https://preview.example.com/task-123"
```

---

## 6. Масштабування

### Фаза 1: Single Worker (MVP)
- 1 worker process
- PostgreSQL для queue
- WebSocket для real-time
- Puppeteer для screenshots

### Фаза 2: Multiple Workers
- N worker processes
- `SELECT FOR UPDATE SKIP LOCKED` для task claiming
- Redis pub/sub для broadcasting між workers
- Shared file system (NFS) або object storage (S3)

### Фаза 3: Isolated Sandboxes
- Docker containers per task
- Firecracker VMs для повної ізоляції
- E2B.dev або Modal.com як managed sandbox
- Паралельне виконання кількох задач

---

## 7. Ключові уроки з OpenClaw

| Урок | Деталі |
|------|--------|
| **Promise chain > queue library** | Для серіалізації в single process достатньо `chain = chain.then(...)` |
| **Schema normalization** | Різні AI провайдери мають різні вимоги до tool schemas. Нормалізуй перед відправкою |
| **Sealing pattern** | Для typing/progress indicators: "seal" після завершення щоб late callbacks не restart |
| **Followup queue** | Якщо юзер відправляє нове повідомлення поки агент працює — ставь в чергу, не переривай |
| **Error recovery layers** | 1) Retry, 2) Fallback model, 3) Session reset, 4) User notification |
| **Session-per-key** | Ізоляція conversations через composite key |
| **Factory pattern для tools** | Lazy initialization з контекстом — tool створюється тільки коли потрібен |
| **Human delay** | 800-2500ms між блоками відповіді — юзер краще сприймає |
| **AbortSignal everywhere** | Пробрасуй AbortSignal через весь стек для graceful cancellation |
| **Config-driven behavior** | Все конфігурується: моделі, tools, queue mode, streaming, timeouts |

---

## 8. Tech Stack Recommendation

| Компонент | Рекомендація | Чому |
|-----------|-------------|------|
| **Framework** | Next.js / Remix | SSR + API routes + WebSocket |
| **Queue** | BullMQ (Redis) або pg-boss (PostgreSQL) | Production-ready, retries, priorities |
| **AI SDK** | Vercel AI SDK або LangChain | Tool calling, streaming, multi-provider |
| **Preview** | Puppeteer + static server | Screenshots + validation |
| **Validation** | Lighthouse CI | Accessibility, performance, SEO |
| **Sandbox** | Docker або E2B.dev | Ізоляція user-generated code |
| **Real-time** | Socket.io або Hono WebSocket | Progress broadcasting |
| **Storage** | S3-compatible | Generated assets |
| **Deploy preview** | Vercel API або Cloudflare Pages API | Instant preview deploys |
| **DB** | PostgreSQL | Tasks, users, sessions |
