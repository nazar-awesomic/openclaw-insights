# Infrastructure Patterns: утиліти, retry, locks, progress

Інсайти з OpenClaw для побудови надійної інфраструктури.

---

## 1. Exponential Backoff з Jitter

**Файл:** `src/infra/backoff.ts:1-29`

```typescript
type BackoffPolicy = {
  initialMs: number;   // Початкова затримка
  maxMs: number;       // Максимальна затримка
  factor: number;      // Множник (зазвичай 2)
  jitter: number;      // Випадковість (0-1)
};

function computeBackoff(policy: BackoffPolicy, attempt: number): number {
  const base = Math.min(
    policy.initialMs * Math.pow(policy.factor, attempt),
    policy.maxMs
  );
  const jitter = base * policy.jitter * Math.random();
  return base + jitter;
}

// Sleep з підтримкою AbortSignal:
async function sleepWithAbort(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(resolve, ms);
    signal?.addEventListener("abort", () => {
      clearTimeout(timer);
      reject(new DOMException("Aborted", "AbortError"));
    }, { once: true });
  });
}
```

**Інсайт:** Завжди використовуй jitter щоб уникнути thundering herd при retry.

---

## 2. Gateway Lock (Process Coordination)

**Файл:** `src/infra/gateway-lock.ts:1-150+`

Тільки один gateway процес може працювати одночасно:

```typescript
// Lock payload:
type LockPayload = {
  pid: number;
  createdAt: string;   // ISO 8601 timestamp (не number!)
  configPath: string;
  startTime?: number;  // optional
};

// Lock directory: os.tmpdir()/openclaw-<uid> (Unix), os.tmpdir()/openclaw (Windows)
// Lock file: gateway.{SHA1(configPath)}.lock — дозволяє кілька конфігів
//
// Перевірка stale lock:
// 1. Перевіряє чи процес живий: process.kill(pid, 0)
// 2. Linux: читає /proc/[pid]/cmdline та /proc/[pid]/stat (start time)
//    — розрізняє reuse PID від оригінального процесу
// 3. Timeout-based acquisition з polling (100ms default interval)
// 4. Три стани: "dead" (видаляє lock), "alive" (чекає), "unknown" (staleness timeout)
```

**Інсайт для твого проекту:** Використовуй distributed locks (Redis SETNX або PostgreSQL advisory locks) для координації worker processes.

---

## 3. Process Management та Signal Handling

### Child Process Bridge

**Файл:** `src/process/child-process-bridge.ts:1-47`

```typescript
function attachChildProcessBridge(child: ChildProcess): { detach: () => void } {
  const signals = process.platform === "win32"
    ? ["SIGTERM", "SIGINT", "SIGBREAK"]
    : ["SIGTERM", "SIGINT", "SIGHUP", "SIGQUIT"];

  const handlers = signals.map(sig => {
    const handler = () => {
      try { child.kill(sig); } catch {}
    };
    process.on(sig, handler);
    return { sig, handler };
  });

  return {
    detach: () => handlers.forEach(h => process.off(h.sig, h.handler)),
  };
}
```

### Gateway Run Loop

**Файл:** `src/cli/gateway-cli/run-loop.ts:1-104`

```typescript
// Graceful shutdown з 5-секундним таймаутом:
process.on("SIGTERM", () => {
  server.stop();
  const forceExitTimer = setTimeout(() => process.exit(1), 5000);
  forceExitTimer.unref(); // Не тримає event loop
});

// In-process restart (SIGUSR1):
process.on("SIGUSR1", () => {
  if (consumeRestartAuthorization()) {
    server.restart(); // Перезапускає без зупинки процесу
  }
});

// Forever-loop з підтримкою restart:
while (true) {
  const lock = await acquireGatewayLock();
  await server.start();
  // Якщо restart — повторюємо цикл
  // Якщо shutdown — виходимо
}
```

---

## 4. Platform-Specific Daemon

**Файл:** `src/daemon/service.ts:1-150+`

```typescript
interface GatewayService {
  install(opts: InstallOpts): Promise<void>;
  uninstall(): Promise<void>;
  stop(): Promise<void>;
  restart(): Promise<void>;
  isLoaded(): Promise<boolean>;
  readCommand(): Promise<string>;
  readRuntime(): Promise<RuntimeInfo>;
}

// Платформо-залежна реалізація:
function resolveGatewayService(): GatewayService {
  if (process.platform === "darwin") return new LaunchAgentService();  // launchd
  if (process.platform === "linux") return new SystemdService();       // systemd
  if (process.platform === "win32") return new ScheduledTaskService(); // schtasks
}
```

---

## 5. Error Handling

**Файл:** `src/infra/errors.ts:1-55`

```typescript
// Type guards для різних форм помилок:
function extractErrorCode(err: unknown): string | number | undefined;
function isErrno(err: unknown): err is NodeJS.ErrnoException;
function hasErrnoCode(err: unknown, code: string): boolean;

// Форматування помилок (обробляє всі типи):
function formatErrorMessage(err: unknown): string {
  if (err instanceof Error) return err.message;
  if (typeof err === "string") return err;
  if (typeof err === "number" || typeof err === "boolean") return String(err);
  if (typeof err === "bigint") return String(err);
  return JSON.stringify(err);
}

// Спеціальна обробка для INVALID_CONFIG:
function formatUncaughtError(err: unknown): string;
```

### CLI Error Pattern:

**Файл:** `src/cli/cli-utils.ts:1-65`

```typescript
// withManager pattern:
async function withManager<T>(
  fn: () => Promise<T>,
  onClose: () => void,
  onError: (err: unknown) => void,
): Promise<T> {
  try {
    return await fn();
  } catch (err) {
    onError(err);
    throw err;
  } finally {
    onClose();
  }
}
```

---

## 6. Configuration Management

**Файл:** `src/config/paths.ts:1-275`

```typescript
// Каскад пошуку конфіга:
// 1. OPENCLAW_CONFIG_PATH env var
// 2. Existing candidates (legacy paths)
// 3. Canonical path: ~/.openclaw/openclaw.json

// Config I/O:
// - JSON5 parsing (коментарі дозволені)
// - SHA256 hashing для change detection
// - Backup rotation (5 бекапів за замовчуванням)
// - .env файли завантажуються автоматично
// - Shell env fallback з timeout

// Config override cascade:
// Environment vars → config file → defaults
```

---

## 7. Logging System

**Файл:** `src/logging/subsystem.ts:1-80+`

```typescript
type SubsystemLogger = {
  trace: LogFn;
  debug: LogFn;
  info: LogFn;
  warn: LogFn;
  error: LogFn;
  fatal: LogFn;
  raw: LogFn;
  child: (subsystem: string) => SubsystemLogger;
};

// Кольори підсистем — hash-based cycling:
const COLORS = ["cyan", "green", "yellow", "blue", "magenta", "red"];
function getSubsystemColor(name: string): string {
  const hash = hashString(name);
  return COLORS[hash % COLORS.length];
}

// Override для конкретних підсистем:
const OVERRIDES = { "gmail-watcher": "blue" };
```

**File Logging:**

```typescript
// Default log dir: /tmp/openclaw
// Uses tslog with file transport
// Rolling rotation: 24-hour age max
// Config-driven: logging.level, logging.file
```

---

## 8. Progress Reporting

**Файл:** `src/cli/progress.ts:1-231`

```typescript
type ProgressReporter = {
  setLabel(label: string): void;
  setPercent(percent: number): void;
  tick(): void;
  done(): void;
};

// Стратегії відображення:
// 1. OSC progress protocol (terminal integration)
// 2. @clack/prompts spinner (TTY)
// 3. Line-based rendering з ANSI line clearing
// 4. Log mode з 250ms throttling (non-TTY)
// 5. Noop reporter (disabled/nested)

// Delayed start:
const reporter = createProgress({
  delayMs: 300, // Не показувати для швидких операцій
  mode: "indeterminate",
});
```

---

## 9. Terminal UI

### Palette (Кольори)

**Файл:** `src/terminal/palette.ts:1-12`

```typescript
const palette = {
  accent: "#FF5A2D",      // Orange-red
  accentBright: "#FF7A3D",
  accentDim: "#D14A22",
  info: "#FF8A5B",
  success: "#2FBF71",
  warn: "#FFB020",
  error: "#E23D2D",
  muted: "#8B7F77",
};
```

### Table Rendering

**Файл:** `src/terminal/table.ts:1-100+`

```typescript
type Column = {
  key: string;
  header: string;
  align: "left" | "right" | "center";
  minWidth?: number;
  maxWidth?: number;
  flex?: boolean; // Авторозмір
};

// ANSI-aware wrapping:
// Ніколи не розбиває SGR (Select Graphic Rendition) sequences
// Ніколи не розбиває OSC-8 (hyperlink) sequences
// visibleWidth() рахує тільки видимі символи
```

### Safe Stream Writer

**Файл:** `src/terminal/stream-writer.ts:1-69`

```typescript
class SafeStreamWriter {
  // Graceful handling of EPIPE та EIO errors
  // Before-write hooks
  // Broken pipe notifications
  // isClosed() check
}
```

---

## 10. Outbound Message Delivery

**Файл:** `src/infra/outbound/deliver.ts:1-120+`

```typescript
type OutboundSendDeps = {
  sendWhatsApp: SendFn;
  sendTelegram: SendFn;
  sendDiscord: SendFn;
  sendSlack: SendFn;
  // ...
};

type DeliveryResult = {
  channel: string;
  messageId: string;
  // + channel-specific fields
};

// Plugin architecture:
// Делегує до ChannelOutboundAdapter per channel
// Message chunking по параграфам або markdown
// Configurable limits per channel
```

---

## 11. Патерни для твого проекту

### Worker Process Architecture:

```typescript
// worker.ts — фоновий процес обробки задач
import { computeBackoff, sleepWithAbort } from "./infra/backoff";

class TaskWorker {
  private abortController = new AbortController();

  async start() {
    process.on("SIGTERM", () => this.shutdown());
    process.on("SIGINT", () => this.shutdown());

    while (!this.abortController.signal.aborted) {
      const task = await this.pollTask();

      if (!task) {
        await sleepWithAbort(1000, this.abortController.signal);
        continue;
      }

      await this.processTask(task);
    }
  }

  async processTask(task: Task) {
    const maxRetries = 3;
    const backoffPolicy = { initialMs: 1000, maxMs: 30000, factor: 2, jitter: 0.2 };

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        await this.executeTask(task);
        return; // Success
      } catch (err) {
        if (attempt === maxRetries - 1) throw err;

        const delay = computeBackoff(backoffPolicy, attempt);
        await sleepWithAbort(delay, this.abortController.signal);
      }
    }
  }

  async shutdown() {
    this.abortController.abort();
    // Wait for current task to finish (timeout 5s)
    setTimeout(() => process.exit(1), 5000).unref();
  }
}
```

### Progress Broadcasting:

```typescript
// Для WebSocket клієнтів:
class TaskProgressBroadcaster {
  sendProgress(taskId: string, update: ProgressUpdate) {
    const clients = this.getSubscribers(taskId);
    for (const client of clients) {
      try {
        client.send(JSON.stringify(update));
      } catch {
        this.removeSubscriber(client); // EPIPE-like handling
      }
    }
  }
}

type ProgressUpdate =
  | { type: "status"; status: TaskStatus }
  | { type: "tool_call"; tool: string; args: unknown }
  | { type: "preview"; html: string }
  | { type: "streaming"; text: string }
  | { type: "error"; message: string }
  | { type: "completed"; result: TaskResult };
```
