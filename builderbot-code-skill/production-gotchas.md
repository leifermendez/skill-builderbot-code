# BuilderBot Production Gotchas

> Critical bugs discovered in real production implementations that don't show error messages but silently break flows.

This document consolidates hard-won lessons from BuilderBot bots running in production, handling real users and real conversations. These gotchas are **silent killers** — they don't throw errors, they just make your bot stop responding, leaving you wondering what went wrong.

**Source**: Patterns extracted from production medical appointment booking systems, e-commerce bots, and customer support implementations.

---

## ⚠️ Gotcha #2: gotoFlow() + capture Bug

### The Problem

When you use `gotoFlow()` to navigate to another flow, and that target flow **immediately** starts with `addAnswer(..., { capture: true })`, the bot **will not receive the user's response**.

**Symptoms:**
- User is prompted for input (e.g., "What's your name?")
- User types a response
- Bot ignores it completely
- No error in console
- Flow doesn't advance

This is another silent killer — the flow appears to work (the prompt is sent), but the capture never fires.

---

### Why It Happens

When `gotoFlow()` executes, BuilderBot immediately switches to the target flow and starts executing its chain. If the first step in that flow is a `capture: true`, BuilderBot sets up the capture handler **but the user's next message arrives before the handler is fully registered**.

**Timing issue:**
1. `gotoFlow(targetFlow)` executes
2. Target flow's first `addAnswer` with `capture: true` registers the handler
3. User sends message ⚡ (arrives too fast)
4. Handler not ready yet → message is ignored
5. Handler becomes ready → but message already passed

This is a **race condition** between handler registration and message arrival.

---

### Wrong vs Right

#### ❌ Wrong — Capture Immediately After gotoFlow

```typescript
// welcome.flow.ts
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAction(async (ctx, { gotoFlow }) => {
        const { nameFlow } = await import('./name.flow')
        return gotoFlow(nameFlow)  // Navigate to name flow
    })

// name.flow.ts
const nameFlow = addKeyword('NAME_FLOW_INTERNAL')
    .addAnswer('What is your name?', { capture: true }, async (ctx, { state }) => {
        // 💀 This capture will NEVER fire if we came from gotoFlow
        await state.update({ name: ctx.body })
    })
```

**What happens:**
1. User types "hello"
2. Bot navigates to `nameFlow` via `gotoFlow()`
3. Bot sends "What is your name?"
4. User types "Jorge"
5. Message "Jorge" is **ignored** 💀
6. Bot stuck waiting for input that already arrived

**User experience**: "I already told you my name! Is this bot deaf?"

---

#### ✅ Right — Buffer with addAction or Prompt in Source Flow

**Solution 1: Capture data BEFORE gotoFlow**

```typescript
// welcome.flow.ts
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAnswer('Hi! What is your name?', { capture: true }, async (ctx, { state, gotoFlow }) => {
        // ✅ Capture happens here, BEFORE navigation
        await state.update({ name: ctx.body })
        
        const { confirmationFlow } = await import('./confirmation.flow')
        return gotoFlow(confirmationFlow)
    })

// confirmation.flow.ts
const confirmationFlow = addKeyword('CONFIRMATION_INTERNAL')
    .addAction(async (ctx, { state, flowDynamic }) => {
        // ✅ Just read from state — no capture needed
        const { name } = state.getMyState()
        await flowDynamic(`Nice to meet you, ${name}!`)
    })
```

**Key difference**: Capture happens **before** `gotoFlow()`, not after.

---

**Solution 2: Use addAction as buffer in target flow**

```typescript
// welcome.flow.ts
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAction(async (ctx, { gotoFlow }) => {
        const { nameFlow } = await import('./name.flow')
        return gotoFlow(nameFlow)
    })

// name.flow.ts
const nameFlow = addKeyword('NAME_FLOW_INTERNAL')
    .addAction(async (ctx, { flowDynamic }) => {
        // ✅ Buffer action — gives handler time to register
        await flowDynamic('Let me ask you a few questions...')
    })
    .addAnswer('What is your name?', { capture: true }, async (ctx, { state }) => {
        // ✅ Now the capture will work because there was a step before it
        await state.update({ name: ctx.body })
    })
```

**Key difference**: The `addAction` before `addAnswer` gives BuilderBot time to fully register the capture handler.

---

**Solution 3: Send prompt from source flow, capture in target**

```typescript
// welcome.flow.ts
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAnswer('Hi! What is your name?')  // ✅ Prompt sent here (no capture)
    .addAction(async (ctx, { gotoFlow }) => {
        const { nameFlow } = await import('./name.flow')
        return gotoFlow(nameFlow)
    })

// name.flow.ts
const nameFlow = addKeyword('NAME_FLOW_INTERNAL')
    .addAnswer(null, { capture: true }, async (ctx, { state }) => {
        // ✅ Capture works because prompt was sent in previous flow
        await state.update({ name: ctx.body })
    })
```

**Key difference**: Prompt and capture are separated by the `gotoFlow()` call, giving time for handler registration.

---

### How to Debug

If your captures aren't firing after `gotoFlow()`:

1. **Check if the target flow starts with `capture: true`** immediately
2. **Add a console.log inside the capture callback**:
   ```typescript
   .addAnswer('Name?', { capture: true }, async (ctx, { state }) => {
       console.log('[DEBUG] Capture fired! Body:', ctx.body)  // Does this log appear?
       await state.update({ name: ctx.body })
   })
   ```
3. **If the log never appears** → you hit the race condition
4. **The fix**: Use one of the three solutions above (capture before gotoFlow, buffer action, or separate prompt)

---

### Real-World Example

**Scenario**: Multi-step appointment booking flow

**Bug**: After implementing modular flows (welcome → appointment type → date selection), users would select "First visit" but when asked "What date?", their responses were ignored.

**Root cause**:
```typescript
// appointmentType.flow.ts
.addAnswer('Select type: 1. First visit  2. Follow-up', { capture: true }, async (ctx, { gotoFlow }) => {
    const type = ctx.body === '1' ? 'first_visit' : 'follow_up'
    await state.update({ appointmentType: type })
    
    const { dateFlow } = await import('./date.flow')
    return gotoFlow(dateFlow)  // Navigate to date selection
})

// date.flow.ts (WRONG)
const dateFlow = addKeyword('DATE_FLOW_INTERNAL')
    .addAnswer('What date works for you?', { capture: true }, async (ctx, { state }) => {
        // 💀 Never fires — race condition
        await state.update({ selectedDate: ctx.body })
    })
```

**Fix**: Added buffer action
```typescript
// date.flow.ts (FIXED)
const dateFlow = addKeyword('DATE_FLOW_INTERNAL')
    .addAction(async (ctx, { flowDynamic }) => {
        // ✅ Buffer — gives time for capture handler to register
        await flowDynamic('Perfect! Let me check available dates...')
    })
    .addAnswer('What date works for you?', { capture: true }, async (ctx, { state }) => {
        // ✅ Now it works!
        await state.update({ selectedDate: ctx.body })
    })
```

**Result**: Date captures now work 100% of the time.

---

### Summary

**Rule**: Never start a target flow with `capture: true` immediately after `gotoFlow()`. Always buffer with at least one non-capturing step.

**Why**: Race condition — the capture handler isn't fully registered when the user's message arrives.

**Solutions**:
1. Capture data **before** calling `gotoFlow()`
2. Add `addAction` buffer in target flow before the capture
3. Send prompt in source flow, capture in target flow

**Best practice**: Keep related captures in the same flow to avoid this issue entirely.

---

**Previous**: [Gotcha #1: Flow Terminal Bug](#gotcha-1-flow-terminal-bug)  
**Next**: [Gotcha #3: Guard Pattern for Duplicate Flows](#) (coming soon)
