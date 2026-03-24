---
name: builderbot-code-skill
description: >-
  Expert assistant for BuilderBot (v1.4.0) — a TypeScript/JavaScript framework for building multi-platform chatbots (WhatsApp, Telegram, Instagram, Email, etc.).
  Use when creating or editing flows (addKeyword, addAnswer, addAction), wiring EVENTS, managing per-user state or globalState, configuring providers (Baileys, Meta, Telegram, Evolution, etc.) or databases (Mongo, Postgres, MySQL, JSON), implementing REST API endpoints (handleCtx, httpServer), debugging flow control (gotoFlow, endFlow, fallBack, idle, capture, flowDynamic), or handling blacklist logic.
  Architecture: Provider + Database + Flow.
---

# BuilderBot Code Skill

## Operate

1. **Identify the stack** — confirm provider package, database adapter, and flow entrypoints before writing code.
2. **Type the callback signature correctly**:
   - `ctx: BotContext` → `{ body, from, name, host, ...platform fields }`
   - `methods: BotMethods` → `{ flowDynamic, gotoFlow, endFlow, fallBack, state, globalState, blacklist, provider, database, extensions }`
3. **Enforce flow-control semantics** — these three MUST be `return`ed:
   - `return gotoFlow(flow)` · `return endFlow('msg')` · `return fallBack('msg')`
   - These MUST be `await`ed: `await flowDynamic(...)` · `await state.update(...)` · `await globalState.update(...)`
4. **Prefer small, composable flows** — one responsibility per flow file.
5. **Keep secrets in env vars** — never hardcode tokens, keys, or credentials.

## Critical Rules (common bugs)

| Bug | Wrong | Right |
|-----|-------|-------|
| Missing `return` on flow control | `gotoFlow(flow)` | `return gotoFlow(flow)` |
| Missing `await` on state/dynamic | `state.update({...})` | `await state.update({...})` |
| Circular ESM import | `import { flow } from './flow'` inside `gotoFlow` | `return gotoFlow(require('./flow').myFlow)` or dynamic `import()` |
| `idle` without `capture` | `{ idle: 5000 }` | `{ capture: true, idle: 5000 }` |

## Flow Chain

```typescript
addKeyword(keywords, options?)        // ActionPropertiesKeyword: capture, idle, buttons, media, delay, regex, sensitive
  .addAnswer(message, options?, cb?, childFlows?)
  .addAction(cb)
// cb: async (ctx: BotContext, methods: BotMethods) => void
```

## State Quick Reference

```typescript
// Per-user
await state.update({ key: value })
state.get('key')            // dot notation supported: 'user.profile.name'
state.getMyState()
state.clear()

// Global (shared across all users)
await globalState.update({ key: value })
globalState.get('key')
globalState.getAllState()
```

## Debug Checklist

- Flow not switching → `return gotoFlow(...)`
- Session not ending → `return endFlow(...)`
- Fallback not repeating → `return fallBack(...)`
- State not saved → `await state.update(...)`
- Idle not firing → add `capture: true` alongside `idle`
- EVENTS flow not triggering → verify provider maps the event payload to `EVENTS.*`
- Circular import crash → use `require()` or dynamic `import()` inside the callback

## References

- Code patterns (scaffold, capture, gotoFlow, media, buttons, idle, flowDynamic, fallBack, REST, EVENTS): [patterns.md](patterns.md)
- Provider configs, database configs, TypeScript types: [providers.md](providers.md)
