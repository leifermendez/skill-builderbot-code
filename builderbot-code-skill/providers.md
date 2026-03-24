# BuilderBot — Providers, Databases & Types

## Providers (14 available)

| Provider | Package | Platform |
|----------|---------|----------|
| Baileys | `@builderbot/provider-baileys` | WhatsApp Web |
| Meta | `@builderbot/provider-meta` | WhatsApp Business API |
| Twilio | `@builderbot/provider-twilio` | SMS / WhatsApp |
| Telegram | `@builderbot/provider-telegram` | Telegram |
| Instagram | `@builderbot/provider-instagram` | Instagram DMs |
| Facebook | `@builderbot/provider-facebook-messenger` | Messenger |
| Email | `@builderbot/provider-email` | IMAP / SMTP |
| Evolution | `@builderbot/provider-evolution-api` | Multi-channel |
| GoHighLevel | `@builderbot/provider-gohighlevel` | CRM |
| Gupshup | `@builderbot/provider-gupshup` | WhatsApp Cloud |
| Sherpa | `@builderbot/provider-sherpa` | WhatsApp Web |
| Venom | `@builderbot/provider-venom` | WhatsApp Web |
| WPPConnect | `@builderbot/provider-wppconnect` | WhatsApp Web |
| WebWhatsApp | `@builderbot/provider-web-whatsapp` | WhatsApp Web |

---

## Provider Configs

### Baileys

```typescript
import { BaileysProvider } from '@builderbot/provider-baileys'

const adapterProvider = createProvider(BaileysProvider, {
    gifPlayback: false,
    usePairingCode: false,
    phoneNumber: null,
    browser: ['Windows', 'Chrome', 'Chrome 114.0.5735.198'],
    useBaileysStore: true,
    groupsIgnore: true,
    readStatus: false,
})
```

### Meta (WhatsApp Business API)

```typescript
import { MetaProvider } from '@builderbot/provider-meta'

const adapterProvider = createProvider(MetaProvider, {
    jwtToken: process.env.META_JWT_TOKEN,
    numberId: process.env.META_NUMBER_ID,
    verifyToken: process.env.META_VERIFY_TOKEN,
    version: 'v18.0',
})
```

### Email

```typescript
import { EmailProvider } from '@builderbot/provider-email'

const adapterProvider = createProvider(EmailProvider, {
    imap: { host: 'imap.gmail.com', port: 993, secure: true, auth: { user: process.env.EMAIL_USER, pass: process.env.EMAIL_PASS } },
    smtp: { host: 'smtp.gmail.com', port: 465, secure: true, auth: { user: process.env.EMAIL_USER, pass: process.env.EMAIL_PASS } },
})
```

### Evolution API

```typescript
import { EvolutionProvider } from '@builderbot/provider-evolution-api'

const adapterProvider = createProvider(EvolutionProvider, {
    apiKey: process.env.EVOLUTION_API_KEY,
    baseURL: 'http://localhost:8080',
    instanceName: 'my-bot',
})
```

---

## Databases (5 available)

| Database | Package | Use Case |
|----------|---------|----------|
| Memory | Built-in (`@builderbot/bot`) | Testing / prototyping |
| JSON | `@builderbot/database-json` | Development |
| MongoDB | `@builderbot/database-mongo` | Production |
| PostgreSQL | `@builderbot/database-postgres` | Production |
| MySQL | `@builderbot/database-mysql` | Production |

---

## Database Configs

### MemoryDB (built-in)

```typescript
import { MemoryDB } from '@builderbot/bot'
const adapterDB = new MemoryDB()
```

### JsonFileDB

```typescript
import { JsonFileDB } from '@builderbot/database-json'
const adapterDB = new JsonFileDB({ filename: 'db.json', debounceTime: 100 })
```

### MongoDB

```typescript
import { MongoAdapter } from '@builderbot/database-mongo'
const adapterDB = new MongoAdapter({
    dbUri: process.env.MONGO_URI,
    dbName: 'botdb',
})
```

### PostgreSQL

```typescript
import { PostgreSQLAdapter } from '@builderbot/database-postgres'
const adapterDB = new PostgreSQLAdapter({
    host: process.env.PG_HOST,
    user: process.env.PG_USER,
    database: process.env.PG_DATABASE,
    password: process.env.PG_PASSWORD,
    port: Number(process.env.PG_PORT) || 5432,
})
```

### MySQL

```typescript
import { MysqlAdapter } from '@builderbot/database-mysql'
const adapterDB = new MysqlAdapter({
    host: process.env.MYSQL_HOST,
    user: process.env.MYSQL_USER,
    database: process.env.MYSQL_DATABASE,
    password: process.env.MYSQL_PASSWORD,
    port: 3306,
})
```

---

## Community Plugins

### Telegram Plugin

```bash
pnpm install @builderbot-plugins/telegram
```

```typescript
import { TelegramProvider } from '@builderbot-plugins/telegram'

const adapterProvider = createProvider(TelegramProvider, {
    token: process.env.TELEGRAM_BOT_TOKEN,
})
```

Flows work identically to WhatsApp — same `addKeyword`, `addAnswer`, `addAction`, `EVENTS`, and state APIs.

---

### OpenAI Agents Plugin

Routes users to specialized AI agents based on intent.

```bash
npm install @builderbot-plugins/openai-agents
```

```typescript
import { EmployeesClass } from '@builderbot-plugins/openai-agents'

const employees = new EmployeesClass({
    apiKey: process.env.OPENAI_API_KEY,
    model: 'gpt-3.5-turbo-16k',
    temperature: 0,
})

employees.employees([
    {
        name: 'SALES_AGENT',
        description: "I'm Rob, I help with purchases and product questions.",
        flow: salesFlow,
    },
    {
        name: 'SUPPORT_AGENT',
        description: "I'm Sara, I resolve technical support issues.",
        flow: supportFlow,
    },
])

// Entry point — uses EVENTS.WELCOME
const welcomeFlow = addKeyword(EVENTS.WELCOME)
    .addAction(async (ctx, ctxFn) => {
        const pluginAi = ctxFn.extensions.employeesAddon
        const result = await pluginAi.determine(ctx.body)
        if (!result?.employee) {
            return ctxFn.flowDynamic("Sorry, I didn't understand. How can I help?")
        }
        ctxFn.state.update({ answer: result.answer })
        pluginAi.gotoFlow(result.employee, ctxFn)
    })

// Each agent flow reads the pre-computed answer from state
const salesFlow = addKeyword(EVENTS.ACTION)
    .addAction(async (_, { state, flowDynamic }) => {
        return flowDynamic(state.getMyState().answer)
    })

await createBot(
    { flow: createFlow([welcomeFlow, salesFlow, supportFlow]), provider, database },
    { extensions: { employeesAddon: employees } }
)
```

---

## Docker Deployment

### Dockerfile

```dockerfile
FROM node:21-alpine3.18 as builder
RUN corepack enable && corepack prepare pnpm@latest --activate
ENV PNPM_HOME=/usr/local/bin
WORKDIR /app
COPY package*.json pnpm-lock.yaml ./
COPY . .
RUN pnpm i

FROM builder as deploy
COPY --from=builder /app/src ./src
COPY --from=builder /app/package.json /app/pnpm-lock.yaml ./
RUN pnpm install
CMD ["pnpm", "start"]
```

```bash
# Build
docker build . -t builderbot:latest

# Run (exposes port 3008, mounts sessions, auto-restarts)
docker run \
  --name "bot" \
  --env PORT=3008 \
  --env OPENAI_API_KEY="your_key" \
  -p 3008:3008/tcp \
  -v "$(pwd)/bot_sessions:/app/bot_sessions:rw" \
  --cap-add SYS_ADMIN \
  --restart always \
  builderbot:latest
```

> `--cap-add SYS_ADMIN` is required by Baileys (Chromium-based providers). Remove it for Meta/Twilio/Evolution.

---

## Key TypeScript Types

### BotContext

```typescript
type BotContext = {
    name?: string
    host?: { phone: string; [key: string]: any }
    body: string        // message content
    from: string        // sender phone / ID
    idleFallBack?: boolean  // true when idle timeout fires
    [key: string]: any
}
```

### BotMethods

```typescript
type BotMethods<P = {}, B = {}> = {
    flowDynamic: (messages: string | string[] | FlowDynamicMessage[], opts?: { delay: number }) => Promise<void>
    gotoFlow:    (flow: TFlow<P>, step?: number) => Promise<void>
    endFlow:     (message?: string) => void
    fallBack:    (message?: string) => void
    state:       BotStateStandAlone
    globalState: BotStateGlobal
    blacklist:   DynamicBlacklist
    provider:    P
    database:    B
    extensions:  Record<string, any>
}
```

### ActionPropertiesKeyword

```typescript
type ActionPropertiesKeyword = {
    idle?:      number    // timeout ms — requires capture: true
    buttons?:   Button[]  // interactive buttons
    media?:     string    // URL or local path
    capture?:   boolean   // wait for user reply
    delay?:     number    // ms before sending
    regex?:     boolean   // treat keyword as regex
    sensitive?: boolean   // case-sensitive match (default: false)
}
```

### FlowDynamicMessage

```typescript
type FlowDynamicMessage = {
    body?:    string
    buttons?: Button[]
    delay?:   number
    media?:   string
}
```

### State Types

```typescript
type BotStateStandAlone = {
    update:     (props: Record<string, any>) => Promise<void>
    getMyState: <K = any>() => Record<string, K>
    get:        <K = any>(prop: string) => K   // dot notation: 'user.profile.name'
    clear:      () => void
}

type BotStateGlobal = {
    update:      (props: Record<string, any>) => Promise<void>
    getAllState:  () => Record<string, any>
    get:         <K = any>(prop: string) => K
    clear:       () => void
}
```

### GeneralArgs (createBot options)

```typescript
type GeneralArgs = {
    blackList?:  string[]
    delay?:      number
    globalState?: Record<string, any>
    extensions?:  Record<string, any>
    queue?: {
        timeout:          number   // default: 50000
        concurrencyLimit: number   // default: 15
    }
}
```

### DynamicBlacklist

```typescript
type DynamicBlacklist = {
    add:     (phoneNumbers: string | string[]) => string[]
    remove:  (phoneNumber: string) => void
    checkIf: (phoneNumber: string) => boolean
    getList: () => string[]
}
```
