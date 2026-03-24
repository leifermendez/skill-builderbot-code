# BuilderBot — Code Patterns

## Basic Bot Scaffold

```typescript
import { createBot, createProvider, createFlow, addKeyword, MemoryDB } from '@builderbot/bot'
import { BaileysProvider as Provider } from '@builderbot/provider-baileys'

const PORT = process.env.PORT ?? 3008

const welcomeFlow = addKeyword<Provider, MemoryDB>(['hi', 'hello'])
    .addAnswer('Welcome!')

const main = async () => {
    const adapterFlow = createFlow([welcomeFlow])
    const adapterProvider = createProvider(Provider)
    const adapterDB = new MemoryDB()

    const { handleCtx, httpServer } = await createBot({
        flow: adapterFlow,
        provider: adapterProvider,
        database: adapterDB,
    })

    httpServer(+PORT)
}

main()
```

---

## Capture + State

```typescript
const registerFlow = addKeyword<Provider, Database>(utils.setEvent('REGISTER'))
    .addAnswer('Your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ name: ctx.body })
    })
    .addAnswer('Your age?', { capture: true }, async (ctx, { state }) => {
        await state.update({ age: ctx.body })
    })
    .addAction(async (_, { flowDynamic, state }) => {
        const { name, age } = state.getMyState()
        await flowDynamic(`Thanks ${name}! Age: ${age}`)
    })
```

---

## addKeyword — Regex & Sensitive

```typescript
// Regex match — pass { regex: true }
const GMAIL_REGEX = /(\w+)@gmail\.com/g
const gmailFlow = addKeyword(GMAIL_REGEX, { regex: true })
    .addAnswer('Thanks for the gmail address!')

// Sensitive — exact full-message match only (default is substring match)
const buyFlow = addKeyword('buy', { sensitive: true })
    .addAnswer('What do you want to buy?')
```

---

## addAnswer — Line Breaks & Delay

```typescript
// Array of strings = single message with line breaks
const infoFlow = addKeyword('info')
    .addAnswer(['Line 1', 'Line 2', 'Line 3'])          // one message, multi-line
    .addAnswer('This arrives after 2 s', { delay: 2000 })

// Callback runs AFTER the message is sent
const greetFlow = addKeyword('hello')
    .addAnswer('Hi! Do you know 4+4?', null, async (_, { flowDynamic }) => {
        await flowDynamic(`Total: ${4 + 4}`)
    })
```

---

## addAction — Capture as First Arg

`addAction` also accepts `{ capture: true }` as its first argument (no preceding `addAnswer` needed):

```typescript
const nameFlow = addKeyword(['hello', 'hi'])
    .addAction(async (_, { flowDynamic }) => {
        await flowDynamic('Hi! What is your name?')
    })
    .addAction({ capture: true }, async (ctx, { state, flowDynamic }) => {
        await state.update({ name: ctx.body })
        await flowDynamic(`Nice to meet you, ${ctx.body}!`)
    })
```

---

## Flow Navigation (gotoFlow)

```typescript
const docFlow = addKeyword<Provider, Database>('doc')
    .addAnswer('Continue? *yes*', { capture: true }, async (ctx, { gotoFlow, flowDynamic }) => {
        if (ctx.body.toLowerCase().includes('yes')) {
            return gotoFlow(registerFlow)   // MUST return
        }
        await flowDynamic('Thanks!')
    })
```

**ESM circular-import guard** — use dynamic `import()` inside the callback (project is `"type": "module"`, `require` is not available):

```typescript
const { registerFlow } = await import('./register.flow')
return gotoFlow(registerFlow)
```

**Real-world pattern** — route registered vs new users via an API check:

```typescript
import { EVENTS } from '@builderbot/bot'
import registeredFlow from './flows/registeredFlow'
import newUserFlow from './flows/newUserFlow'

const welcomeFlow = addKeyword(EVENTS.WELCOME)
    .addAction(async (ctx, { gotoFlow, state }) => {
        const res = await fetch('https://my.app/api/check', {
            method: 'POST',
            body: JSON.stringify({ phone: ctx.from }),
            headers: { 'Content-Type': 'application/json' },
        })
        const { registered } = await res.json()
        if (registered) return gotoFlow(registeredFlow)
        await state.update({ registered: false })
        return gotoFlow(newUserFlow)
    })
```

---

## Media

```typescript
import { join } from 'path'

const mediaFlow = addKeyword<Provider, Database>('samples')
    .addAnswer('Image:', { media: join(process.cwd(), 'assets', 'sample.png') })
    .addAnswer('Video:', { media: 'https://example.com/video.mp4' })
    .addAnswer('PDF:', { media: 'https://example.com/doc.pdf' })
```

---

## UX Patterns — Conversational Best Practices

### 1. Always tell the user what to type

```typescript
.addAnswer('What is your email? _(e.g. you@example.com)_', { capture: true }, async (ctx, { fallBack, state }) => {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(ctx.body)) {
        return fallBack('That doesn\'t look right. Please send a valid email (e.g. you@example.com):')
    }
    await state.update({ email: ctx.body })
})
```

### 2. Cancel / exit at any capture step

```typescript
.addAnswer('Enter your address _(or type *cancel* to abort)_', { capture: true }, async (ctx, { endFlow, state }) => {
    if (ctx.body.toLowerCase() === 'cancel') {
        return endFlow('No problem! Send *menu* whenever you\'re ready.')
    }
    await state.update({ address: ctx.body })
})
```

### 3. Confirm before committing irreversible actions

```typescript
.addAction(async (_, { flowDynamic, state }) => {
    const { product, qty, address } = state.getMyState()
    await flowDynamic([
        `*Order summary:*`,
        `• Product: ${product}`,
        `• Qty: ${qty}`,
        `• Ship to: ${address}`,
        '',
        'Reply *yes* to confirm or *no* to cancel.',
    ])
})
.addAction({ capture: true }, async (ctx, { gotoFlow, endFlow }) => {
    if (ctx.body.toLowerCase() === 'yes') return gotoFlow(confirmOrderFlow)
    return endFlow('Order cancelled. Send *menu* to start over.')
})
```

### 4. Break long content into steps (no wall of text)

```typescript
// Bad — one huge message
.addAnswer('Here is everything you need to know about our return policy...[300 words]')

// Good — chunk it
.addAnswer('*Return policy* — 3 key points:')
.addAnswer('1. Returns accepted within 30 days of delivery.', { delay: 800 })
.addAnswer('2. Item must be unused and in original packaging.', { delay: 800 })
.addAnswer('3. Refunds processed in 5–7 business days.', { delay: 800 })
.addAnswer('Need help with a return? Reply *yes* or *no*.')
```

### 5. Always close with a clear ending

```typescript
.addAction(async (ctx, { flowDynamic, endFlow }) => {
    await flowDynamic(`Done! Your order has been placed. You'll get a confirmation shortly.`)
    return endFlow()
})
```

---

## Buttons — DO NOT USE

`buttons` is unreliable across providers (Baileys, Meta, Telegram, etc.) and must **NOT** be used.
Use a text-based numbered menu with `capture: true` instead:

```typescript
const menuFlow = addKeyword<Provider, Database>('menu')
    .addAnswer(
        ['Please choose an option:', '1. Track order', '2. Contact support', '3. FAQ'],
        { capture: true },
        async (ctx, { gotoFlow, fallBack }) => {
            if (ctx.body === '1') return gotoFlow(trackFlow)
            if (ctx.body === '2') return gotoFlow(supportFlow)
            if (ctx.body === '3') return gotoFlow(faqFlow)
            return fallBack('Invalid option. Please reply with 1, 2, or 3.')
        }
    )
```

---

## Idle Timeout

`idle` requires `capture: true`. When the timeout fires the flow expires automatically — do **NOT** check `ctx.idleFallBack`.

```typescript
const orderFlow = addKeyword<Provider, Database>('order')
    .addAnswer(
        'What to order? _(you have 60 s to reply)_',
        { capture: true, idle: 60000 },   // flow auto-expires after 60 s
        async (ctx, { state }) => {
            await state.update({ order: ctx.body })
        }
    )
```

---

## flowDynamic Variants

```typescript
const dynamicFlow = addKeyword<Provider, Database>('search')
    .addAction(async (ctx, { flowDynamic }) => {
        await flowDynamic('Searching...')                          // string
        await flowDynamic(['Result 1', 'Result 2'])               // string[]
        await flowDynamic([{                                       // FlowDynamicMessage[]
            body: 'Found!',
            delay: 500,
            media: 'https://example.com/result.png',
        }])
    })
```

---

## FallBack Validation

```typescript
const emailFlow = addKeyword<Provider, Database>('email')
    .addAnswer(
        'Enter your email:',
        { capture: true },
        async (ctx, { fallBack, flowDynamic }) => {
            if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(ctx.body)) {
                return fallBack('Invalid email. Try again:')      // MUST return
            }
            await flowDynamic(`Saved: ${ctx.body}`)
        }
    )
```

---

## EVENTS

`WELCOME` fires when no keyword matches the user's text (catch-all fallback).

```typescript
import { EVENTS, addKeyword } from '@builderbot/bot'

// WELCOME — default catch-all
const welcomeFlow = addKeyword(EVENTS.WELCOME)
    .addAnswer('Hey! How can I help you?')

// MEDIA — image or video received; use provider.saveFile() to persist
const mediaFlow = addKeyword(EVENTS.MEDIA)
    .addAnswer('Got your media!', null, async (ctx, { provider }) => {
        const localPath = await provider.saveFile(ctx, { path: './uploads' })
        console.log('Saved to:', localPath)
    })

// DOCUMENT — file/PDF received
const docFlow = addKeyword(EVENTS.DOCUMENT)
    .addAnswer("Got your document!", null, async (ctx, { provider }) => {
        const localPath = await provider.saveFile(ctx, { path: './uploads' })
    })

// VOICE_NOTE — audio message
const voiceFlow = addKeyword(EVENTS.VOICE_NOTE)
    .addAnswer('Give me a second to process your audio!', null, async (ctx, { provider }) => {
        const localPath = await provider.saveFile(ctx, { path: './uploads' })
    })

// LOCATION — coordinates from WhatsApp native location share
const locationFlow = addKeyword(EVENTS.LOCATION)
    .addAnswer('Got your location!', null, async (ctx) => {
        const lat = ctx.message.locationMessage.degreesLatitude
        const lng = ctx.message.locationMessage.degreesLongitude
        console.log({ lat, lng })
    })

// Register all:
const adapterFlow = createFlow([welcomeFlow, mediaFlow, docFlow, voiceFlow, locationFlow])
```

Available events: `WELCOME`, `MEDIA`, `LOCATION`, `DOCUMENT`, `VOICE_NOTE`, `ORDER`, `TEMPLATE`, `CALL`

> **Note**: Location must be sent from WhatsApp's native share-location button — external map links are not parsed.

---

## REST Endpoints

The HTTP server is powered by [polka](https://github.com/lukeed/polka) (lightweight express-like). `handleCtx` injects the bot instance into each route handler.

```typescript
const { handleCtx, httpServer } = await createBot({ flow, provider: adapterProvider, database: adapterDB })

// Send a text message
adapterProvider.server.post('/v1/messages', handleCtx(async (bot, req, res) => {
    const { number, message } = req.body
    await bot.sendMessage(number, message, {})
    return res.end('sent')
}))

// Send message + media (URL or local path)
adapterProvider.server.post('/v1/messages', handleCtx(async (bot, req, res) => {
    const { number, message, media } = req.body
    await bot.sendMessage(number, message, { media })   // media = URL or file path
    return res.end('sent')
}))

// Trigger a named flow for a specific user
// Flow must use utils.setEvent('EVENT_NAME') as keyword
adapterProvider.server.post('/v1/register', handleCtx(async (bot, req, res) => {
    const { number, name } = req.body
    await bot.dispatch('EVENT_REGISTER', { from: number, name })
    return res.end('triggered')
}))

// Manage blacklist (prevents bot from responding to a number)
adapterProvider.server.post('/v1/blacklist', handleCtx(async (bot, req, res) => {
    const { number, intent } = req.body
    if (intent === 'add')    bot.blacklist.add(number)
    if (intent === 'remove') bot.blacklist.remove(number)
    res.writeHead(200, { 'Content-Type': 'application/json' })
    return res.end(JSON.stringify({ status: 'ok', number, intent }))
}))

httpServer(+PORT)
```

Curl examples:

```bash
# Send message
curl -X POST http://localhost:3008/v1/messages -d number="34000000" -d message="Hello!"

# Trigger flow
curl -X POST http://localhost:3008/v1/register -d number="34000000" -d name="Joe"

# Blacklist
curl -X POST http://localhost:3008/v1/blacklist -d number="34000000" -d intent="add"
```

---

## GlobalState (shared across users)

```typescript
.addAction(async (ctx, { globalState }) => {
    await globalState.update({ totalMessages: (globalState.get('totalMessages') ?? 0) + 1 })
    const total = globalState.get<number>('totalMessages')
    await flowDynamic(`Bot has handled ${total} messages total.`)
})
```

---

## Blacklist

```typescript
.addAction(async (ctx, { blacklist }) => {
    blacklist.add(ctx.from)           // block
    blacklist.remove(ctx.from)        // unblock
    blacklist.checkIf(ctx.from)       // boolean
    blacklist.getList()               // string[]
})
```

---

## Extensions (dependency injection)

Pass services into every flow callback via `extensions`:

```typescript
import { ai } from './services/ai'

await createBot(
    { flow, provider, database },
    { extensions: { ai } }
)

// Inside any callback:
.addAction(async (ctx, { extensions }) => {
    const answer = await extensions.ai.ask(ctx.body)
    await flowDynamic(answer)
})
```

---

## Queue Config (high-traffic bots)

Default handles ~15 concurrent users. Increase for high-load scenarios:

```typescript
await createBot(
    { flow, provider, database },
    {
        queue: {
            timeout: 20000,        // max ms per async handler (default 50000)
            concurrencyLimit: 50,  // parallel users processed (default 15)
        },
    }
)
```

---

## Modular Project Structure

> **Important**: BuilderBot projects use `"type": "module"` in `package.json` and `"module": "ES2022"` in tsconfig.
> **NEVER** use `require()` — it is not available. Use ESM `import` / `export` and dynamic `await import()` for circular cases.

Reference: [BuilderBot — Modularize Flows](https://www.builderbot.app/en/showcases/modularize)

### Recommended layout

```
my-bot/
├── src/
│   ├── app.ts                  # entry — wires provider, database, flow
│   ├── provider/
│   │   └── index.ts            # export const provider = createProvider(...)
│   ├── database/
│   │   └── index.ts            # export const database = new MongoAdapter(...)
│   ├── flows/
│   │   ├── index.ts            # barrel — export const flow = createFlow([...])
│   │   ├── welcome.flow.ts
│   │   └── register.flow.ts
│   └── services/
│       └── ai.ts               # export const ai = new MyAiService()
├── assets/
├── .env
├── package.json                # "type": "module"
├── tsconfig.json
└── rollup.config.js
```

### flows/index.ts (barrel file)

```typescript
import { createFlow } from '@builderbot/bot'
import { welcomeFlow } from './welcome.flow'
import { registerFlow } from './register.flow'

export const flow = createFlow([welcomeFlow, registerFlow])
```

### app.ts (entry point)

```typescript
import { createBot } from '@builderbot/bot'
import { flow } from './flows'
import { database } from './database'
import { provider } from './provider'
import { ai } from './services/ai'

const main = async () => {
    await createBot({ flow, provider, database }, { extensions: { ai } })
    provider.initHttpServer(+(process.env.PORT ?? 3008))
}
main()
```

### Cross-flow navigation (avoiding circular imports)

When flow A navigates to flow B and B also navigates to A, do NOT import both at the top level. Use dynamic `import()` inside the callback:

```typescript
// welcome.flow.ts
.addAction(async (ctx, { gotoFlow }) => {
    const { orderFlow } = await import('./order.flow')
    return gotoFlow(orderFlow)
})
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "outDir": "./dist",
    "skipLibCheck": true
  }
}
```

### rollup.config.js

```javascript
import typescript from 'rollup-plugin-typescript2'
export default {
    input: 'src/app.ts',
    output: { file: 'dist/app.js', format: 'esm' },
    plugins: [typescript()],
}
```
