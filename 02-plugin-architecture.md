# Plugin/Extension Architecture

Інсайти з OpenClaw для побудови розширюваної системи.

---

## 1. Plugin Contract (Контракт плагіна)

**Файл:** `src/plugins/types.ts:229-283`

OpenClaw має простий, але потужний API для плагінів:

```typescript
// Плагін — це або об'єкт з register(), або просто функція:
type PluginModule =
  | { id: string; name: string; register(api: PluginApi): void }
  | ((api: PluginApi) => void);

// API яке отримує плагін при реєстрації:
type PluginApi = {
  id: string;
  config: Config;
  runtime: PluginRuntime;
  logger: Logger;

  // Методи реєстрації:
  registerTool(factory, opts?): void;
  registerHook(event, handler): void;
  registerHttpHandler(handler): void;
  registerHttpRoute({ path, handler }): void;
  registerChannel(registration): void;
  registerGatewayMethod(method, handler): void;
  registerCli(registrar, opts?): void;
  registerService(service): void;
  registerCommand(definition): void;
};
```

**Інсайт:** Один API для всіх типів розширень. Плагін сам вирішує що реєструвати.

---

## 2. Plugin Discovery та Loading

**Файл:** `src/plugins/discovery.ts:115-150`, `src/plugins/loader.ts:169-453`

### Джерела плагінів (від найвищого пріоритету до найнижчого):
1. **Config** — шляхи з `config.plugins.load.paths` (найвищий пріоритет)
2. **Workspace** — `<workspaceDir>/.openclaw/extensions/`
3. **Global** — `~/.openclaw/extensions/`
4. **Bundled** — вбудовані в репозиторій (найнижчий пріоритет)

> **Важливо:** Перший знайдений плагін з даним `id` виграє. Config-плагіни перевизначають bundled.

### Процес завантаження:

```
Discover candidates (scan directories)
    ↓
Load manifests (openclaw.plugin.json)
    ↓
Create registry (prepare API factory)
    ↓
Dynamic import via jiti (supports .ts without build)
    ↓
Resolve exports (default/function/object)
    ↓
Validate config (JSON Schema)
    ↓
Call register(api)
    ↓
Cache registry (by workspace hash)
```

**Ключовий трюк — jiti для TypeScript без білду:**

```typescript
const jiti = createJiti(import.meta.url, {
  interopDefault: true,
  extensions: [".ts", ".tsx", ".mts", ".cts", ".js", ".mjs", ".cjs"],
  alias: { "openclaw/plugin-sdk": resolvedSdkPath },
});
const mod = jiti(candidate.source); // Імпортує .ts напряму
```

---

## 3. Plugin Manifest

**Файл:** `src/plugins/manifest.ts:10-21`

Кожен плагін має `openclaw.plugin.json`:

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string" }
    },
    "required": ["apiKey"]
  }
}
```

Плюс metadata в `package.json`:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "discord",
      "label": "Discord",
      "order": 10
    },
    "install": {
      "npmSpec": "@openclaw/discord",
      "localPath": "extensions/discord"
    }
  }
}
```

---

## 4. Plugin Registry (Реєстр)

**Файл:** `src/plugins/registry.ts:124-138`

```typescript
type PluginRegistry = {
  plugins: PluginRecord[];
  tools: PluginToolRegistration[];
  hooks: PluginHookRegistration[];
  channels: PluginChannelRegistration[];
  providers: PluginProviderRegistration[];
  gatewayHandlers: GatewayRequestHandlers;
  httpHandlers: PluginHttpRegistration[];
  httpRoutes: PluginHttpRouteRegistration[];
  cliRegistrars: PluginCliRegistration[];
  services: PluginServiceRegistration[];
  commands: PluginCommandRegistration[];
  diagnostics: PluginDiagnostic[];
};
```

**Глобальний стан реєстру:**

```typescript
// src/plugins/runtime.ts:1-50
const state = { registry: null, key: null };

export function setActivePluginRegistry(registry, cacheKey?) {
  state.registry = registry;
  state.key = cacheKey;
}

export function requireActivePluginRegistry(): PluginRegistry {
  if (!state.registry) state.registry = createEmptyRegistry();
  return state.registry;
}
```

---

## 5. Tool Factory Pattern

**Файл:** `src/plugins/types.ts:68-76`

Tools створюються через factory для lazy initialization з контекстом:

```typescript
type ToolFactory = (ctx: ToolContext) => Tool | Tool[] | null;

type ToolContext = {
  config?: Config;
  workspaceDir?: string;
  agentId?: string;
  sessionKey?: string;
  messageChannel?: string;
  sandboxed?: boolean;  // Чи в sandbox режимі
};

// Реєстрація:
api.registerTool(
  (ctx) => {
    if (ctx.sandboxed) return null; // Не доступний в sandbox
    return createMyTool(api);
  },
  { optional: true }  // Може бути вимкнений через tool-policy
);
```

**Інсайт:** Factory pattern дозволяє:
- Умовну реєстрацію (sandboxed, permissions)
- Lazy initialization (tool створюється тільки коли потрібен)
- Контекстно-залежні tools (різна поведінка для різних агентів)

---

## 6. Hooks System (Lifecycle Events)

**Файл:** `src/plugins/types.ts:298-312`

```typescript
type HookName =
  | "before_agent_start"   // Модифікація system prompt
  | "agent_end"            // Після завершення агента
  | "before_compaction"    // Перед стисненням контексту
  | "after_compaction"     // Після стиснення контексту
  | "message_received"     // Вхідне повідомлення
  | "message_sending"      // Перед відправкою
  | "message_sent"         // Після відправки
  | "before_tool_call"     // Перед виконанням tool
  | "after_tool_call"      // Після виконання tool
  | "tool_result_persist"  // Збереження результату tool
  | "session_start"        // Початок сесії
  | "session_end"          // Кінець сесії
  | "gateway_start"        // Старт gateway
  | "gateway_stop";        // Зупинка gateway

// Реєстрація:
api.on("before_agent_start", (event, ctx) => {
  return { systemPrompt: "Additional instructions..." };
});

api.on("message_received", (event, ctx) => {
  // event: { from, content, timestamp, metadata }
  // ctx: { channelId, accountId, conversationId }
});
```

---

## 7. Service Registration (Фонові сервіси)

**Файл:** `src/plugins/types.ts:218-222`

```typescript
type PluginService = {
  id: string;
  start(ctx: ServiceContext): Promise<void>;
  stop?(ctx: ServiceContext): Promise<void>;
};

api.registerService({
  id: "my-background-service",
  start: async (ctx) => {
    // ctx.stateDir — директорія для стану
    startPolling();
  },
  stop: async () => {
    stopPolling();
  },
});
```

---

## 8. Channel Plugin (Adapter Pattern)

**Файл:** `src/channels/plugins/types.plugin.ts:48-84`

Канал — це набір адаптерів:

```typescript
type ChannelPlugin = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;

  // ~20 optional адаптерів:
  config: ConfigAdapter;        // Конфігурація акаунтів
  onboarding?: OnboardingAdapter;
  pairing?: PairingAdapter;     // Підключення юзерів
  outbound?: OutboundAdapter;   // Відправка повідомлень
  gateway?: GatewayAdapter;     // HTTP/WebSocket handlers
  streaming?: StreamingAdapter; // Стрімінг конфіг
  threading?: ThreadingAdapter; // Треди
  mentions?: MentionAdapter;    // @mentions
  actions?: ActionAdapter;      // Реакції, кнопки
  status?: StatusAdapter;       // Здоров'я/діагностика
  commands?: CommandAdapter;    // Native commands
  // ...
};
```

**Capabilities декларують що канал підтримує:**

```typescript
type ChannelCapabilities = {
  chatTypes: (ChatType | "thread")[];  // ChatType = "direct" | "group" | "channel"
  polls: boolean;
  reactions: boolean;
  edit: boolean;
  unsend: boolean;
  reply: boolean;
  effects?: boolean;         // Анімації/ефекти
  threads: boolean;
  media: boolean;
  nativeCommands: boolean;
  blockStreaming: boolean;
  groupManagement?: boolean; // Управління групами
};
```

---

## 9. Dependency Injection Pattern

**Файл:** `src/cli/deps.ts:9-39`

```typescript
type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  // ...
};

function createDefaultDeps(): CliDeps {
  return {
    sendMessageWhatsApp,
    sendMessageTelegram,
    sendMessageDiscord,
    // ...
  };
}

// Використання — deps за замовчуванням або для тестів:
function someCommand(deps: CliDeps = createDefaultDeps()) {
  deps.sendMessageDiscord(/* ... */);
}
```

**Інсайт:** Простий DI без фреймворків. Deps передаються через параметри, defaults через factory function.

---

## 10. Патерн для твого проекту

### Рекомендована архітектура плагінів:

```typescript
// plugin-api.ts
interface PluginApi {
  registerGenerator(generator: PageGenerator): void;
  registerTemplate(template: Template): void;
  registerTool(tool: AgentTool): void;
  registerHook(event: HookEvent, handler: HookHandler): void;
  registerPostProcessor(processor: PostProcessor): void;
}

// Приклад плагіна — TailwindCSS генератор:
export default function register(api: PluginApi) {
  api.registerGenerator({
    id: "tailwind",
    name: "Tailwind CSS Generator",
    generate: async (prompt, context) => {
      return generateTailwindPage(prompt, context);
    },
  });

  api.registerPostProcessor({
    id: "tailwind-purge",
    process: async (html) => purgeTailwindCSS(html),
  });
}

// Приклад плагіна — Analytics:
export default function register(api: PluginApi) {
  api.registerHook("page_generated", async (event) => {
    await trackGeneration(event.taskId, event.metrics);
  });

  api.registerHook("page_deployed", async (event) => {
    await injectAnalytics(event.pageUrl);
  });
}
```

### Hooks для landing page builder:

```typescript
type HookEvent =
  | "task_created"        // Нова задача в черзі
  | "task_started"        // Агент почав працювати
  | "code_generated"      // Код згенерований
  | "preview_ready"       // Preview готовий
  | "validation_failed"   // Перевірка не пройшла
  | "page_deployed"       // Сторінка задеплоєна
  | "user_feedback"       // Юзер дав фідбек
  | "iteration_start"     // Початок ітерації правок
  | "iteration_end";      // Кінець ітерації правок
```
