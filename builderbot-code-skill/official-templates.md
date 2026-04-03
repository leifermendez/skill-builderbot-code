# BuilderBot Official CLI Templates

> Complete reference extracted from `@builderbot/cli@1.4.1` package — the exact templates that `npm create builderbot@latest` generates.

This document provides the authoritative configuration for BuilderBot projects, extracted directly from the official CLI package. Use these templates as the reference when scaffolding new projects or updating existing ones.

---

## 📦 Template Matrix

The official CLI offers **120+ templates** combining:

**Languages**: TypeScript (`base-ts-*`) | JavaScript (`base-js-*`)

**Providers**:
- `baileys` - WhatsApp Web (Baileys library) — most common
- `meta` - Meta Business API
- `twilio` - Twilio API
- `email` - Email provider
- `evolution-api` - Evolution API
- `facebook-messenger` - Facebook Messenger
- `gohighlevel` - GoHighLevel integration
- `instagram` - Instagram Direct
- `sherpa` - Sherpa API
- `wppconnect` - WPPConnect library
- `gupshup` - Gupshup API

**Databases**:
- `memory` - In-memory (MemoryDB) — fastest, data lost on restart
- `mongo` - MongoDB — most popular for production
- `mysql` - MySQL
- `postgres` - PostgreSQL
- `json` - JSON file storage — simple, file-based persistence

**Template naming**: `base-{ts|js}-{provider}-{database}`

**Examples**:
- `base-ts-baileys-memory` - TypeScript + Baileys + MemoryDB (quickest to start)
- `base-ts-baileys-mongo` - TypeScript + Baileys + MongoDB (production ready)
- `base-js-meta-postgres` - JavaScript + Meta + PostgreSQL

---

## 🚀 Complete Official Template (base-ts-baileys-memory)

### app.ts

```typescript
import { join } from 'path'
import { createBot, createProvider, createFlow, addKeyword, utils } from '@builderbot/bot'
import { MemoryDB as Database } from '@builderbot/bot'
import { BaileysProvider as Provider } from '@builderbot/provider-baileys'

const PORT = process.env.PORT ?? 3008

const discordFlow = addKeyword<Provider, Database>('doc').addAnswer(
    ['You can see the documentation here', '📄 https://builderbot.app/docs \n', 'Do you want to continue? *yes*'].join(
        '\n'
    ),
    { capture: true },
    async (ctx, { gotoFlow, flowDynamic }) => {
        if (ctx.body.toLocaleLowerCase().includes('yes')) {
            return gotoFlow(registerFlow)
        }
        await flowDynamic('Thanks!')
        return
    }
)

const welcomeFlow = addKeyword<Provider, Database>(['hi', 'hello', 'hola'])
    .addAnswer(`🙌 Hello welcome to this *Chatbot*`)
    .addAnswer(
        [
            'I share with you the following links of interest about the project',
            '👉 *doc* to view the documentation',
        ].join('\n'),
        { delay: 800, capture: true },
        async (ctx, { fallBack }) => {
            if (!ctx.body.toLocaleLowerCase().includes('doc')) {
                return fallBack('You should type *doc*')
            }
            return
        },
        [discordFlow]
    )

const registerFlow = addKeyword<Provider, Database>(utils.setEvent('REGISTER_FLOW'))
    .addAnswer(`What is your name?`, { capture: true }, async (ctx, { state }) => {
        await state.update({ name: ctx.body })
    })
    .addAnswer('What is your age?', { capture: true }, async (ctx, { state }) => {
        await state.update({ age: ctx.body })
    })
    .addAction(async (_, { flowDynamic, state }) => {
        await flowDynamic(`${state.get('name')}, thanks for your information!: Your age: ${state.get('age')}`)
    })

const fullSamplesFlow = addKeyword<Provider, Database>(['samples', utils.setEvent('SAMPLES')])
    .addAnswer(`💪 I'll send you a lot files...`)
    .addAnswer(`Send image from Local`, { media: join(process.cwd(), 'assets', 'sample.png') })
    .addAnswer(`Send video from URL`, {
        media: 'https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExYTJ0ZGdjd2syeXAwMjQ4aWdkcW04OWlqcXI3Ynh1ODkwZ25zZWZ1dCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/LCohAb657pSdHv0Q5h/giphy.mp4',
    })
    .addAnswer(`Send audio from URL`, { media: 'https://cdn.freesound.org/previews/728/728142_11861866-lq.mp3' })
    .addAnswer(`Send file from URL`, {
        media: 'https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf',
    })

const main = async () => {
    const adapterFlow = createFlow([welcomeFlow, registerFlow, fullSamplesFlow])
    
    // If you experience ERRO AUTH issues, check the latest WhatsApp version at:
    // https://wppconnect.io/whatsapp-versions/
    // Example: version "2.3000.1035824857-alpha" -> [2, 3000, 1035824857]
    const adapterProvider = createProvider(Provider, 
        { version: [2, 3000, 1035824857] } 
    )
    const adapterDB = new Database()

    const { handleCtx, httpServer } = await createBot({
        flow: adapterFlow,
        provider: adapterProvider,
        database: adapterDB,
    })

    // HTTP API Endpoints
    adapterProvider.server.post(
        '/v1/messages',
        handleCtx(async (bot, req, res) => {
            const { number, message, urlMedia } = req.body
            await bot.sendMessage(number, message, { media: urlMedia ?? null })
            return res.end('sended')
        })
    )

    adapterProvider.server.post(
        '/v1/register',
        handleCtx(async (bot, req, res) => {
            const { number, name } = req.body
            await bot.dispatch('REGISTER_FLOW', { from: number, name })
            return res.end('trigger')
        })
    )

    adapterProvider.server.post(
        '/v1/samples',
        handleCtx(async (bot, req, res) => {
            const { number, name } = req.body
            await bot.dispatch('SAMPLES', { from: number, name })
            return res.end('trigger')
        })
    )

    adapterProvider.server.post(
        '/v1/blacklist',
        handleCtx(async (bot, req, res) => {
            const { number, intent } = req.body
            if (intent === 'remove') bot.blacklist.remove(number)
            if (intent === 'add') bot.blacklist.add(number)

            res.writeHead(200, { 'Content-Type': 'application/json' })
            return res.end(JSON.stringify({ status: 'ok', number, intent }))
        })
    )

    adapterProvider.server.get(
        '/v1/blacklist/list',
        handleCtx(async (bot, req, res) => {
            const blacklist = bot.blacklist.getList()
            res.writeHead(200, { 'Content-Type': 'application/json' })
            return res.end(JSON.stringify({ status: 'ok', blacklist }))
        })
    )

    httpServer(+PORT)
}

main()
```

---

## 🌐 HTTP API Endpoints (Built-in)

The official template includes a complete REST API for programmatic control:

| Endpoint | Method | Description | Request Body |
|----------|--------|-------------|--------------|
| `/v1/messages` | POST | Send message to any WhatsApp number | `{ number, message, urlMedia? }` |
| `/v1/register` | POST | Trigger registration flow for a number | `{ number, name }` |
| `/v1/samples` | POST | Trigger samples flow (media examples) | `{ number, name }` |
| `/v1/blacklist` | POST | Add/remove number from blacklist | `{ number, intent: "add" \| "remove" }` |
| `/v1/blacklist/list` | GET | Get all blacklisted numbers | N/A |

### Usage Examples

**Send a message programmatically:**
```bash
curl -X POST http://localhost:3008/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5491112345678",
    "message": "Hello from API!",
    "urlMedia": "https://example.com/image.jpg"
  }'
```

**Trigger a flow from external system:**
```bash
# Trigger registration flow when user signs up on website
curl -X POST http://localhost:3008/v1/register \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5491112345678",
    "name": "Jorge"
  }'
```

**Add number to blacklist:**
```bash
curl -X POST http://localhost:3008/v1/blacklist \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5491112345678",
    "intent": "add"
  }'
```

**Get blacklist:**
```bash
curl http://localhost:3008/v1/blacklist/list
# Response: {"status":"ok","blacklist":["5491112345678",...]}
```

---

## ⚙️ Configuration Files

### package.json

```json
{
  "name": "my-builderbot",
  "version": "1.0.0",
  "description": "",
  "main": "dist/app.js",
  "type": "module",
  "scripts": {
    "start": "node ./dist/app.js",
    "lint": "eslint . --no-ignore",
    "dev": "npm run lint && nodemon --signal SIGKILL ./src/app.ts",
    "build": "npx rollup -c"
  },
  "keywords": [],
  "dependencies": {
    "@builderbot/bot": "latest",
    "@builderbot/provider-baileys": "latest"
  },
  "devDependencies": {
    "@types/node": "^25.0.0",
    "typescript-eslint": "^8.0.0",
    "eslint": "^9.0.0",
    "eslint-plugin-builderbot": "latest",
    "rollup": "^4.10.0",
    "nodemon": "^3.1.11",
    "rollup-plugin-typescript2": "^0.36.0",
    "tsx": "^4.7.1",
    "typescript": "^5.4.3"
  },
  "author": "",
  "license": "ISC"
}
```

**Key notes**:
- Dependencies use `"latest"` to always get newest compatible version
- `"type": "module"` enables ESM imports
- `eslint-plugin-builderbot` provides BuilderBot-specific linting rules

**For MongoDB projects**, add:
```json
"@builderbot/database-mongo": "latest"
```

---

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "declaration": false,
    "declarationMap": false,
    "moduleResolution": "node",
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "sourceMap": false,
    "outDir": "./dist",
    "baseUrl": "./",
    "rootDir": "./",
    "incremental": true,
    "skipLibCheck": true,
    "paths": {
      "~/*": ["./src/*"]
    }
  },
  "include": [
    "**/*.js",
    "**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.spec.ts",
    "**e2e**",
    "**mock**"
  ]
}
```

**Key features**:
- **ES2022 target** (newer than ES2020) — enables latest JavaScript features
- **Path alias** `~/*` maps to `src/*` — cleaner imports: `import { flow } from '~/flows/flow'`
- **Excludes test files** automatically
- **No source maps** in production (set `sourceMap: true` for debugging)

---

### eslint.config.js

```javascript
import tseslint from 'typescript-eslint'
import builderbot from 'eslint-plugin-builderbot'

export default [
    {
        ignores: ['dist/**', 'node_modules/**', 'rollup.config.js'],
    },
    ...tseslint.configs.recommended,
    {
        plugins: {
            builderbot,
        },
        languageOptions: {
            ecmaVersion: 'latest',
            sourceType: 'module',
        },
        rules: {
            ...builderbot.configs.recommended.rules,
            '@typescript-eslint/no-explicit-any': 'off',
            '@typescript-eslint/no-unused-vars': 'off',
            '@typescript-eslint/ban-ts-comment': 'off',
            'no-unsafe-optional-chaining': 'off',
        },
    },
]
```

**Key features**:
- Uses **ESLint v9+ flat config** format (not `.eslintrc.*`)
- Includes `eslint-plugin-builderbot` for framework-specific rules
- Extends `typescript-eslint` recommended config
- Relaxes some strict rules for rapid prototyping

---

### nodemon.json

```json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ],
  "delay": "3",
  "execMap": {
    "ts": "tsx"
  }
}
```

**Key features**:
- Watches `src/` directory only
- Uses `tsx` (not `ts-node`) — faster, ESM-compatible
- 3ms delay before restart (prevents rapid restarts)
- Ignores test files

---

### .gitignore

```gitignore
dist/*
node_modules
.env
.pnpm-store
*_sessions
*tokens
.wwebjs*

*.log
*qr.png
```

**BuilderBot-specific entries**:
- `*_sessions` - Baileys session data (phone authentication)
- `*tokens` - Auth tokens
- `.wwebjs*` - WPPConnect artifacts
- `*qr.png` - Generated QR codes for pairing

---

### rollup.config.js

```javascript
import typescript from 'rollup-plugin-typescript2';

export default {
    input: 'src/app.ts',
    output: {
        file: 'dist/app.js',
        format: 'esm',
        sourcemap: true
    },
    external: [],
    plugins: [
        typescript({
            tsconfig: './tsconfig.json',
            declaration: false,
            declarationMap: false
        })
    ]
};
```

**Build process**:
```bash
npm run build  # Compiles TypeScript to dist/app.js
npm start      # Runs compiled dist/app.js (production)
```

---

## 📱 WhatsApp Version Pinning

The official template includes WhatsApp version pinning to avoid `ERRO_AUTH` issues:

```typescript
const adapterProvider = createProvider(Provider, 
    { version: [2, 3000, 1035824857] } 
)
```

### Why pin the version?

WhatsApp Web updates frequently. BuilderBot (via Baileys) emulates a specific WhatsApp Web version. If the version is too old, WhatsApp servers reject authentication.

### How to update the version

1. Visit **https://wppconnect.io/whatsapp-versions/**
2. Find the latest **stable** version (e.g., `"2.3000.1035824857-alpha"`)
3. Convert to array format: `[2, 3000, 1035824857]`
4. Update provider config:
   ```typescript
   const adapterProvider = createProvider(Provider, 
       { version: [2, 3000, 1035824857] }  // ← Update here
   )
   ```

**When to update:**
- Bot fails to generate QR code
- Authentication errors (`ERRO_AUTH`)
- After major WhatsApp Web updates

---

## 🛠️ Installation Workflow

### Standard (Linux/Mac)

```bash
npm create builderbot@latest my-bot
cd my-bot
npm install
npm run dev
```

### Windows (CRITICAL)

BuilderBot on Windows has a known dependency resolution issue with npm.

**Mandatory workflow:**
```bash
npm create builderbot@latest my-bot
cd my-bot
npm install           # Step 1: Install with npm
pnpm install          # Step 2: IMMEDIATELY run pnpm (fixes missing deps)
npm run dev
```

**Why?** npm on Windows sometimes fails to resolve transitive dependencies (Baileys, sharp, qrcode-terminal). pnpm catches missing packages.

**If you don't have pnpm:**
```bash
npm install -g pnpm
```

**Verify installation:**
```bash
npm list @builderbot/bot @builderbot/provider-baileys
# Should show versions, not "UNMET DEPENDENCY"
```

---

## 📂 Project Structure

```
my-bot/
├── src/
│   ├── app.ts              # Entry point (main function)
│   └── flows/              # Optional: organize flows in subdirectory
│       ├── welcome.flow.ts
│       ├── menu.flow.ts
│       └── register.flow.ts
├── assets/
│   └── sample.png          # Media files for testing
├── dist/                   # Compiled JavaScript (generated by build)
├── node_modules/
├── .env                    # Environment variables (PORT, MONGO_URI, etc.)
├── .gitignore
├── eslint.config.js
├── nodemon.json
├── package.json
├── rollup.config.js
└── tsconfig.json
```

---

## 🔗 Additional Resources

- **Official CLI Package**: https://www.npmjs.com/package/create-builderbot
- **BuilderBot Docs**: https://builderbot.vercel.app/
- **Baileys (WhatsApp Web API)**: https://github.com/WhiskeySockets/Baileys
- **WhatsApp Version Tracker**: https://wppconnect.io/whatsapp-versions/

---

**Extracted from**: `@builderbot/cli@1.4.1`  
**Template source**: `dist/starters/apps/base-ts-baileys-memory/`  
**Last verified**: 2026-04-03
