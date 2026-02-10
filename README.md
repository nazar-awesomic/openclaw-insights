# OpenClaw Architecture Insights

Архітектурні інсайти з кодової бази OpenClaw для побудови landing page builder (Lovable-подібний проект).

## Файли

| # | Файл | Що описує |
|---|------|-----------|
| 01 | [Agent Orchestration](./01-agent-orchestration.md) | Agentic loop, queue management, followup runs, error recovery, AbortSignal, typing controller |
| 02 | [Plugin Architecture](./02-plugin-architecture.md) | Plugin contract, discovery, loading, registry, hooks, services, channel adapters, DI |
| 03 | [Message Pipeline](./03-message-pipeline.md) | Message flow, context normalization, routing, dispatch, reply dispatcher, streaming, debouncing |
| 04 | [Tool Execution](./04-tool-execution.md) | Tool definition, schema anti-gravity pattern, result format, truncation, canvas, tool policy |
| 05 | [Infrastructure Patterns](./05-infrastructure-patterns.md) | Backoff, locks, process management, error handling, config, logging, progress, terminal UI |
| 06 | [Architecture Overview](./06-architecture-overview.md) | High-level architecture, key decisions, recommended architecture for landing page builder, scaling phases |

## Головні патерни

### 1. Agentic Tool Loop
Модель сама вирішує коли зупинитись. Цикл: model call → tool call → tool result → model call → ... → final response.

### 2. Promise Chain Serialization
`sendChain = sendChain.then(...)` — простий і надійний спосіб серіалізації без зовнішніх queue libraries.

### 3. Schema Anti-Gravity
Різні AI провайдери мають різні вимоги до JSON Schema. Нормалізуй schemas перед відправкою. Ніколи не використовуй `anyOf`/`oneOf` — flatten в enum.

### 4. Plugin-First
Все що можна — через плагіни. Core мінімальний, extension points максимальні.

### 5. Session Isolation
Composite key (`agent:channel:user`) для повної ізоляції conversations.

### 6. Multi-Layer Error Recovery
Retry → Fallback model → Session reset → User notification.

## Quick Start для твого проекту

1. Прочитай [06-architecture-overview.md](./06-architecture-overview.md) — загальна картина
2. Прочитай [01-agent-orchestration.md](./01-agent-orchestration.md) — як оркеструвати агентів
3. Прочитай [04-tool-execution.md](./04-tool-execution.md) — як визначати tools для генерації
4. Реалізуй MVP: Task Queue → Agent Orchestrator → Tool Loop → WebSocket Broadcasting
