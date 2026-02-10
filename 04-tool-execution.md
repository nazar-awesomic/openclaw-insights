# Tool Execution: визначення, виконання, та schema tricks

Інсайти з OpenClaw для побудови системи AI tools.

---

## 1. Tool Definition Pattern

**Файл:** `src/agents/pi-tools.ts:1-453`

### Структура tool:

```typescript
type AgentTool = {
  name: string;           // Унікальне ім'я
  label?: string;         // Людське ім'я
  description: string;    // Опис для AI моделі
  parameters: JSONSchema; // Схема параметрів (TypeBox)
  execute: (
    toolCallId: string,
    params: unknown,
    onUpdate: UpdateCallback,
    ctx: ToolContext,
    signal: AbortSignal,
  ) => Promise<ToolResult>;
};
```

### Збірка tools для агента:

```typescript
// src/agents/pi-tools.ts:115-167
function createTools() {
  return [
    ...codingTools,      // read, write, edit, bash/exec
    ...openclawTools,    // message, browser, canvas, sessions
    ...pluginTools,      // resolvePluginTools(context)
    ...channelTools,     // listChannelAgentTools()
  ];
}
```

---

## 2. Tool Result Format

**Файл:** `src/agents/tools/common.ts:189-243`

```typescript
// Текстовий результат:
function jsonResult(payload: Record<string, unknown>): ToolResult {
  return {
    content: [{ type: "text", text: JSON.stringify(payload) }],
    details: payload,
  };
}

// Результат з зображенням:
function imageResult(params: {
  text: string;
  imageBase64: string;
  mimeType: string;
  details?: Record<string, unknown>;
}): ToolResult {
  return {
    content: [
      { type: "text", text: params.text },
      { type: "image", data: params.imageBase64, mimeType: params.mimeType },
    ],
    details: params.details ?? {},
  };
}
```

---

## 3. Tool Execution Wrapping

**Файл:** `src/agents/pi-tools.ts:436-452`

Tools проходять через кілька обгорток:

```
Raw Tool
    ↓
normalizeToolParameters()      // Фікс schema для різних провайдерів
    ↓
wrapToolWithBeforeToolCallHook()  // Plugin hooks before execution
    ↓
wrapToolWithAbortSignal()      // Підтримка скасування
    ↓
toToolDefinitions()            // Конвертація в формат pi-ai
```

---

## 4. Schema Anti-Gravity Pattern (КРИТИЧНО)

**Файл:** `src/agents/pi-tools.schema.ts:65-175`

### Проблема:
- **OpenAI** відхиляє `anyOf`/`oneOf` на root рівні schema
- **Gemini** відхиляє: `patternProperties`, `additionalProperties`, `format`, `pattern`, `minLength`, `maxLength`, та ще купу JSON Schema keywords
- **Claude** найбільш толерантний, але все одно є quirks

### Рішення — normalizeToolParameters():

```typescript
function normalizeToolParameters(schema: JSONSchema): JSONSchema {
  // 1. Якщо є anyOf/oneOf — flatten в один object
  if (schema.anyOf || schema.oneOf) {
    const variants = schema.anyOf || schema.oneOf;
    const merged = mergeObjectVariants(variants);
    // Об'єднує properties з усіх варіантів
    // Зберігає enum values (action: "present" | "hide" | "navigate")
    return { type: "object", properties: merged };
  }

  // 2. Для кожного property — рекурсивно нормалізувати
  for (const [key, prop] of Object.entries(schema.properties)) {
    schema.properties[key] = normalizeToolParameters(prop);
  }

  return schema;
}
```

### Gemini-specific cleanup:

**Файл:** `src/agents/schema/clean-for-gemini.ts:167-376`

```typescript
function cleanSchemaForGemini(schema: JSONSchema): JSONSchema {
  // Видаляє unsupported keywords:
  delete schema.additionalProperties;
  delete schema.patternProperties;
  delete schema.format;
  delete schema.pattern;
  delete schema.minLength;
  delete schema.maxLength;
  delete schema.minimum;
  delete schema.maximum;
  // ... ще 20+ keywords

  // Resolve $ref з circular reference detection
  // Flatten literal anyOf → flat enum
  // Strip null variants від unions
}
```

### Safe schema creation:

**Файл:** `src/agents/schema/typebox.ts:15-43`

```typescript
// ЗАМІСТЬ:
Type.Union([Type.Literal("a"), Type.Literal("b")])  // ❌ Gemini відхилить

// ВИКОРИСТОВУЙ:
stringEnum(["a", "b"])  // ✅ Працює скрізь

// Реалізація:
function stringEnum(values: string[]) {
  return Type.Unsafe({ type: "string", enum: values });
}

function optionalStringEnum(values: string[]) {
  return Type.Optional(stringEnum(values));
}
```

**Інсайт:** Ніколи не використовуй `Type.Union` в tool schemas. Використовуй `stringEnum` для enum values і `Type.Optional` замість `| null`.

---

## 5. Tool Result Truncation

**Файл:** `src/agents/pi-embedded-subscribe.tools.ts:63-209`

```typescript
// Текст: максимум 8000 символів (безпечно обрізається по UTF-16 boundaries)
// Error messages: максимум 400 символів (тільки перший рядок)
// Images: не включаються в AI контекст (занадто великі)

function sanitizeToolResult(result: ToolResult): SanitizedResult {
  const text = extractToolResultText(result);
  return {
    text: text.slice(0, 8000),
    error: extractToolErrorMessage(result)?.slice(0, 400),
    // images omitted
  };
}
```

---

## 6. Tool-to-Model Adapter

**Файл:** `src/agents/pi-tool-definition-adapter.ts:81-119`

```typescript
function toToolDefinitions(tools: AgentTool[]): ToolDefinition[] {
  return tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    parameters: tool.parameters,
    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
      try {
        // Detect legacy vs current arg signature:
        return await tool.execute(toolCallId, params, onUpdate, ctx, signal);
      } catch (err) {
        return jsonResult({ error: formatErrorMessage(err) });
      }
    },
  }));
}
```

---

## 7. Canvas Tool (UI Generation)

**Файл:** `src/agents/tools/canvas-tool.ts:12-181`

```typescript
// Canvas actions:
type CanvasAction =
  | "present"    // Показати canvas з HTML
  | "hide"       // Сховати canvas
  | "navigate"   // Завантажити URL
  | "eval"       // Виконати JavaScript і повернути результат
  | "snapshot"   // Зробити screenshot (PNG/JPEG)
  | "a2ui_push"  // Push JSONL для рендерингу UI
  | "a2ui_reset" // Очистити UI стан

// Виконання через gateway:
async function executeCanvasAction(action, params) {
  const result = await node.invoke({
    command: `canvas_${action}`,
    params,
    idempotencyKey: crypto.randomUUID(),
  });

  if (action === "snapshot") {
    return imageResult({
      text: "Screenshot captured",
      imageBase64: result.data,
      mimeType: result.mimeType,
    });
  }

  return jsonResult(result);
}
```

**A2UI (Artifact to UI):**

```typescript
// src/canvas-host/a2ui.ts:102-218
// Сервер подає HTML з live-reload injection
// Cross-platform bridge для комунікації:
// iOS:     window.webkit.messageHandlers.openclawCanvasA2UIAction.postMessage()
// Android: window.openclawCanvasA2UIAction.postMessage()
// Web:     window.parent.postMessage()
```

---

## 8. Tool Policy (Контроль доступу)

```typescript
// Глобальний allow/deny:
config.tools.allow = ["read", "write", "browser"];
config.tools.deny = ["bash", "exec"];

// Per-agent:
config.agents["landing-builder"].tools.allow = ["write_html", "screenshot", "deploy"];

// Per-plugin:
config.plugins["my-plugin"].tools.allow = ["custom_tool"];
```

---

## 9. Патерн для твого проекту: Landing Page Tools

```typescript
// tools/write-html.ts
const writeHtmlTool: AgentTool = {
  name: "write_html",
  description: "Write HTML/CSS/JS code for the landing page",
  parameters: Type.Object({
    filename: Type.String({ description: "File path relative to project root" }),
    content: Type.String({ description: "File content" }),
  }),
  execute: async (toolCallId, params) => {
    await writeFile(params.filename, params.content);
    return jsonResult({
      success: true,
      path: params.filename,
      size: params.content.length
    });
  },
};

// tools/preview-screenshot.ts
const screenshotTool: AgentTool = {
  name: "preview_screenshot",
  description: "Take a screenshot of the current landing page preview",
  parameters: Type.Object({
    viewport: optionalStringEnum(["desktop", "tablet", "mobile"]),
  }),
  execute: async (toolCallId, params) => {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    const viewports = {
      desktop: { width: 1440, height: 900 },
      tablet: { width: 768, height: 1024 },
      mobile: { width: 375, height: 812 },
    };
    await page.setViewport(viewports[params.viewport ?? "desktop"]);
    await page.goto(previewUrl);

    const screenshot = await page.screenshot({ encoding: "base64" });
    await browser.close();

    return imageResult({
      text: `Screenshot taken (${params.viewport ?? "desktop"} viewport)`,
      imageBase64: screenshot,
      mimeType: "image/png",
    });
  },
};

// tools/validate-page.ts
const validateTool: AgentTool = {
  name: "validate_page",
  description: "Validate the landing page for accessibility, performance, and responsive design issues",
  parameters: Type.Object({
    checks: Type.Array(
      stringEnum(["accessibility", "performance", "responsive", "seo", "links"])
    ),
  }),
  execute: async (toolCallId, params) => {
    const results = await runLighthouse(previewUrl, params.checks);
    return jsonResult({
      score: results.score,
      issues: results.issues,
      suggestions: results.suggestions,
    });
  },
};

// tools/deploy-preview.ts
const deployPreviewTool: AgentTool = {
  name: "deploy_preview",
  description: "Deploy the landing page to a preview URL for the user to review",
  parameters: Type.Object({
    projectDir: Type.String(),
  }),
  execute: async (toolCallId, params) => {
    const url = await deployToPreview(params.projectDir);
    return jsonResult({ previewUrl: url, deployed: true });
  },
};
```

### Agentic Loop з цими tools:

```
System Prompt: "You are a landing page builder. Generate a beautiful,
responsive landing page based on the user's description. Use the available
tools to write files, preview the result, validate quality, and iterate
until the page meets high standards."

1. Model reads user prompt
2. Calls write_html (index.html with structure)
3. Calls write_html (styles.css with Tailwind)
4. Calls preview_screenshot (desktop)
5. Reviews screenshot → sees layout issues
6. Calls write_html (fix CSS)
7. Calls preview_screenshot (mobile)
8. Calls validate_page ([accessibility, responsive])
9. Calls write_html (fix accessibility issues)
10. Calls deploy_preview
11. Returns final message: "Your landing page is ready at {url}"
```
