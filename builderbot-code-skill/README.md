# BuilderBot Code Skill

> Expert assistant for BuilderBot — the TypeScript/JavaScript framework that makes building production-ready chatbots simple and maintainable.

[![BuilderBot](https://img.shields.io/badge/BuilderBot-v1.4.0-00c200)](https://builderbot.vercel.app/)
[![Skills.sh](https://img.shields.io/badge/skills.sh-official-blue)](https://skills.sh/leifermendez/skill-builderbot-code/builderbot-code-skill)

---

## 🙏 Una Nota de Agradecimiento

Esta skill existe gracias al increíble trabajo de **[Leifer Méndez](https://github.com/leifermendez)** y la comunidad de BuilderBot.

Durante estos años, Leifer ha:
- Creado y mantenido BuilderBot — un framework que democratiza el desarrollo de chatbots
- Construido una comunidad próspera de desarrolladores resolviendo problemas reales
- Documentado patrones, buenas prácticas y casos edge
- Respondido a incontables issues, PRs y preguntas con paciencia y expertise

**Esta skill es nuestra pequeña forma de retribuir.** Consolida los aprendizajes de implementaciones en producción, sesiones de debugging ganadas con esfuerzo, y la sabiduría colectiva de desarrolladores construyendo con BuilderBot en el mundo real.

**Gracias, Leifer.** Tu trabajo ha hecho la diferencia. 🚀

---

## 🎯 What This Skill Does

This skill helps AI coding assistants (Claude, GPT, Copilot, etc.) write **production-grade BuilderBot code** by:

- ✅ Enforcing flow-control semantics (`return gotoFlow()`, `await state.update()`)
- ✅ Catching common bugs before they happen (missing `return`, `capture` without `idle`, etc.)
- ✅ Providing battle-tested patterns from real production bots
- ✅ Guiding UX design with a 10-point conversational checklist
- ✅ Covering all 14 providers and 5 database adapters
- ✅ Including WhatsApp formatting, presence updates, and messaging norms

**Result:** Less debugging, faster iterations, better user experience.

---

## 📦 Installation

### For [Skills.sh](https://skills.sh) (recommended)

```bash
npx skills add https://github.com/leifermendez/skill-builderbot-code --skill builderbot-code-skill
```

### Manual Integration

Add this to your AI assistant's system prompt or project context:

```markdown
Load BuilderBot Code Skill from:
https://github.com/leifermendez/skill-builderbot-code/tree/main/builderbot-code-skill
```

---

## 📚 What's Inside

This skill is organized into three main documents:

### 1. [SKILL.md](./SKILL.md)
**The core guide.** Start here.

- Operational workflow (how to approach BuilderBot development)
- Critical rules and common bugs (table of wrong vs. right patterns)
- Flow chain syntax and state management quick reference
- **UX Review Checklist** — 10 questions to ask before shipping any flow
- WhatsApp formatting and messaging norms
- Module system (ESM), presence updates, debug checklist

**When to use:** Every time you write or review a BuilderBot flow.

---

### 2. [patterns.md](./patterns.md)
**Real-world code patterns.**

Covers:
- Basic bot scaffold
- Capture + state management
- Flow navigation (`gotoFlow`, `endFlow`, `fallBack`)
- Media handling (images, videos, PDFs)
- UX patterns (cancel flows, confirm before commit, break long content)
- `flowDynamic` variants
- EVENTS (`WELCOME`, `MEDIA`, `DOCUMENT`, `LOCATION`, `VOICE_NOTE`)
- REST endpoints (send messages, trigger flows, manage blacklist)
- GlobalState, Blacklist, Extensions
- Queue config for high-traffic bots
- Modular project structure

**When to use:** When you need a specific implementation example (e.g., "How do I handle media?" or "How do I set up a REST endpoint?").

---

### 3. [providers.md](./providers.md)
**Infrastructure reference.**

Covers:
- All 14 providers (Baileys, Meta, Twilio, Telegram, Instagram, Email, Evolution, etc.)
- Provider configs and options
- All 5 databases (Memory, JSON, MongoDB, PostgreSQL, MySQL)
- Community plugins (Telegram, Shopify, OpenAI Agents)
- Docker deployment
- TypeScript types (BotContext, BotMethods, ActionPropertiesKeyword, etc.)

**When to use:** When setting up a new project, switching providers, or configuring infrastructure.

---

## 🚀 Quick Start Example

Here's what the skill helps you write:

```typescript
import { createBot, createProvider, createFlow, addKeyword, MemoryDB } from '@builderbot/bot'
import { BaileysProvider } from '@builderbot/provider-baileys'

const welcomeFlow = addKeyword<BaileysProvider, MemoryDB>(['hello', 'hi'])
    .addAnswer('Hi! What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ name: ctx.body })
    })
    .addAction(async (_, { flowDynamic, state }) => {
        const { name } = state.getMyState()
        await flowDynamic(`Nice to meet you, ${name}! 😊`)
    })

const main = async () => {
    const adapterFlow = createFlow([welcomeFlow])
    const adapterProvider = createProvider(BaileysProvider)
    const adapterDB = new MemoryDB()

    await createBot({
        flow: adapterFlow,
        provider: adapterProvider,
        database: adapterDB,
    })

    adapterProvider.initHttpServer(3008)
}

main()
```

**The skill ensures:**
- ✅ Correct use of `await state.update()`
- ✅ Proper `capture: true` for user input
- ✅ `flowDynamic` with correct syntax
- ✅ Clean conversation flow with greeting and closing

---

## 🛠️ Use Cases

This skill is battle-tested on production bots handling:
- 🏥 Medical appointment booking (multi-step flows, timezone logic, confirmation)
- 🛒 E-commerce sales (product catalog, cart, checkout)
- 📞 Customer support (ticket routing, human handoff)
- 📅 Event scheduling (calendar integration, availability checking)

**Pattern library growing.** Contributions welcome!

---

## 🤝 Contributing

Found a bug? Have a pattern from production? Want to improve the docs?

**We'd love your contribution!**

### How to Contribute

1. **Open an issue** describing the problem or pattern you want to add
2. **Fork the repo** and create a branch
3. **Add your pattern** (use conceptual examples — no client-specific code)
4. **Submit a PR** with a clear description of the problem + solution + impact

**What we're looking for:**
- Production gotchas (bugs that don't show errors but break flows)
- Resilience patterns (retry logic, cache, fallback strategies)
- State management best practices for complex flows
- AI integration patterns (Claude, GPT as intent extractors)
- Real-world UX examples (appointment booking, payments, etc.)

**See something missing?** Open an issue and let's discuss it!

---

## 📖 Related Resources

- [BuilderBot Official Docs](https://builderbot.vercel.app/)
- [BuilderBot GitHub](https://github.com/codigoencasa/bot-whatsapp)
- [BuilderBot Discord Community](https://link.codigoencasa.com/DISCORD)
- [Leifer's Twitter](https://twitter.com/leifermendez)
- [BuilderBot Course](https://app.codigoencasa.com/courses/builderbot?refCode=LEIFER)

---

## 📄 License

MIT — same as BuilderBot.

---

## 💬 Final Words

Building chatbots is hard. BuilderBot makes it easier. This skill makes BuilderBot easier.

If you're using this skill and it saves you time — **give Leifer's [BuilderBot repo](https://github.com/codigoencasa/bot-whatsapp) a star**. ⭐

Better yet, join the [Discord community](https://link.codigoencasa.com/DISCORD) and share what you're building.

**Let's build great conversational experiences together.**

---

**Maintained with ❤️ by the BuilderBot community**
