# BuilderBot Production Gotchas

> Critical bugs discovered in real production implementations that don't show error messages but silently break flows.

This document consolidates hard-won lessons from BuilderBot bots running in production, handling real users and real conversations. These gotchas are **silent killers** — they don't throw errors, they just make your bot stop responding, leaving you wondering what went wrong.

**Source**: Patterns extracted from production medical appointment booking systems, e-commerce bots, and customer support implementations.

---

## ⚠️ Gotcha #3: Guard Pattern for Duplicate Flows

### The Problem

When two or more flows have **overlapping keywords**, both flows trigger simultaneously when a user sends a matching message. This causes:

**Symptoms:**
- Bot sends duplicate responses (same message twice)
- Bot sends conflicting responses (two different flows answering)
- State gets corrupted (two flows updating the same keys)
- Flow logic breaks (user is in two conversations at once)
- Captures fire twice (user input processed by multiple flows)

This is especially problematic when a user is **mid-conversation** in one flow and types a keyword that matches another flow's trigger.

---

### Why It Happens

BuilderBot evaluates **all registered flows** against every incoming message. If multiple flows have keywords that match, **all of them execute**.

**Example scenario:**
```typescript
// Flow 1: General welcome
const welcomeFlow = addKeyword(['hello', 'hi', 'hola'])
    .addAnswer('Hi! How can I help you?')

// Flow 2: Start appointment booking
const appointmentFlow = addKeyword(['hello', 'appointment', 'book'])
    .addAnswer('Let's book an appointment!')
```

When user types `"hello"`:
- ✅ `welcomeFlow` triggers (keyword match)
- ✅ `appointmentFlow` triggers (keyword match)
- 💀 User receives BOTH messages

**More critical scenario** — mid-conversation collision:
```typescript
const bookingFlow = addKeyword(['book', 'appointment'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAnswer('What date?', { capture: true }, async (ctx, { state }) => {
        await state.update({ appointmentDate: ctx.body })
        // User is here, mid-conversation ⬆️
    })

const menuFlow = addKeyword(['menu', 'help', 'hello'])
    .addAnswer('Main menu: 1. Book  2. Cancel  3. Help')
```

If the user is in `bookingFlow` (being asked for a date) and accidentally types `"hello"`:
- ❌ `menuFlow` triggers (keyword match)
- 💀 User is now in TWO flows simultaneously
- 💀 State gets corrupted (both flows writing to it)
- 💀 Next capture might fire in the wrong flow

---

### Wrong vs Right

#### ❌ Wrong — No Guards, Flows Trigger Freely

```typescript
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAnswer('Hi! What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAnswer('Great! How can I help you?')

const appointmentFlow = addKeyword(['hello', 'appointment'])
    .addAnswer('Let's book an appointment!')
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        // 💀 This will also trigger if user says "hello"
        await state.update({ clientName: ctx.body })
    })
```

**What happens when user types "hello":**
1. Both flows trigger simultaneously
2. User receives:
   - "Hi! What's your name?" (from welcomeFlow)
   - "Let's book an appointment!" (from appointmentFlow)
   - "What's your name?" (from appointmentFlow again)
3. User is now in **two conversations at once**
4. State updates happen in both flows → race condition

**User experience**: "Why is this bot asking me twice? Is it broken?"

---

#### ✅ Right — Guard to Prevent Duplicate Triggers

**Solution 1: Check state to see if user is already in a flow**

```typescript
const welcomeFlow = addKeyword(['hello', 'hi'])
    .addAction(async (ctx, { state }) => {
        // ✅ Guard: check if user is already in another flow
        const clientName = await state.get('clientName')
        const slotsCache = await state.get('slotsCache')
        const activeBooking = await state.get('activeBooking')
        
        if (clientName || slotsCache || activeBooking) {
            // User is mid-conversation in another flow — don't interrupt
            console.log('[GUARD] User already in active flow, skipping welcome')
            return  // Exit early, don't trigger this flow
        }
    })
    .addAnswer('Hi! What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
```

**Key principle**: Before starting a flow, check if state contains indicators that another flow is active. If so, exit early.

---

**Solution 2: Use specific state flag for active flow**

```typescript
const appointmentFlow = addKeyword(['book', 'appointment'])
    .addAction(async (ctx, { state }) => {
        // ✅ Guard: check if any flow is active
        const activeFlow = await state.get('activeFlow')
        
        if (activeFlow && activeFlow !== 'appointment') {
            console.log(`[GUARD] User in ${activeFlow} flow, skipping appointment trigger`)
            return  // Don't interrupt other flows
        }
        
        // Mark this flow as active
        await state.update({ activeFlow: 'appointment' })
    })
    .addAnswer('Let's book! What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAction(async (ctx, { state, flowDynamic, endFlow }) => {
        // ... create appointment ...
        
        // ✅ Clear the flag when flow completes
        await state.update({ activeFlow: null })
        return endFlow('Done!')
    })
```

**Key principle**: Use a dedicated `activeFlow` flag. Set it when flow starts, check it before triggering, clear it when flow ends.

---

**Solution 3: Use more specific keywords**

```typescript
// Instead of generic keywords that overlap:
const welcomeFlow = addKeyword(['hello', 'hi'])  // ❌ Too generic

const appointmentFlow = addKeyword(['hello', 'book'])  // ❌ Overlaps with welcome

// Use specific, non-overlapping keywords:
const welcomeFlow = addKeyword(['hello', 'hi', 'hola'])
    // Guard still recommended, but less likely to trigger mid-flow

const appointmentFlow = addKeyword(['book appointment', 'reserve', 'schedule'])
    // More specific — less overlap

const helpFlow = addKeyword(['help', 'menu', 'options'])
    // Distinct from appointment keywords
```

**Key principle**: Design keyword sets to minimize overlap. Use multi-word phrases when possible.

---

### How to Debug

If you're seeing duplicate messages or state corruption:

1. **Check your flow keywords** — do any overlap?
   ```typescript
   // List all your flows and their keywords:
   welcomeFlow: ['hello', 'hi']
   appointmentFlow: ['hello', 'appointment']  // ⚠️ 'hello' overlaps!
   ```

2. **Add debug logs at flow entry**:
   ```typescript
   const myFlow = addKeyword(['hello'])
       .addAction(async (ctx, { state }) => {
           console.log('[DEBUG] myFlow triggered, state:', state.getMyState())
           // Check if state shows user is already in another flow
       })
   ```

3. **Monitor state during user sessions**:
   ```typescript
   .addAction(async (ctx, { state }) => {
       const currentState = state.getMyState()
       console.log('[STATE]', JSON.stringify(currentState, null, 2))
   })
   ```

4. **If multiple flows fire** → add guards as shown above

---

### Real-World Example

**Scenario**: Medical appointment booking bot with main menu

**Bug**: Users in the middle of booking an appointment would accidentally type "hello" or "hi" (common social reflex). This would trigger the welcome flow, interrupting the booking process and causing state corruption.

**Root cause**:
```typescript
// mainMenu.flow.ts
const mainMenuFlow = addKeyword(['hello', 'hi', 'hola', 'menu'])
    .addAction(async (ctx, { state }) => {
        // ❌ No guard — triggers even if user is mid-booking
        await state.update({ slotsCache: null })  // 💀 Wipes booking data!
    })
    .addAnswer('Main menu: ...')

// newPatient.flow.ts
const newPatientFlow = addKeyword(['new patient', 'first visit'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAnswer('Do you have studies?', { capture: true }, async (ctx, { state }) => {
        // User is HERE when they type "hello" out of politeness ⬆️
    })
```

**What happened**:
1. User starts booking: "new patient"
2. Bot asks: "What's your name?"
3. User: "Jorge"
4. Bot asks: "Do you have studies?"
5. User (being polite): "Hello, yes I have them"  ← 💀 Triggers mainMenuFlow
6. `mainMenuFlow` runs → clears `slotsCache` → booking data lost
7. User sees main menu instead of continuing booking

**Fix**: Added guard to mainMenuFlow
```typescript
const mainMenuFlow = addKeyword(['hello', 'hi', 'hola', 'menu'])
    .addAction(async (ctx, { state }) => {
        // ✅ Guard: don't interrupt active bookings
        const clientName = await state.get('clientName')
        const slotsCache = await state.get('slotsCache')
        
        if (clientName || slotsCache) {
            console.log('[GUARD] Active booking detected, skipping main menu')
            return  // Exit early
        }
        
        // Only clear cache if no active booking
        await state.update({ slotsCache: null })
    })
    .addAnswer('Main menu: ...')
```

**Result**: Users can now type "hello" mid-booking without losing their progress.

---

### Best Practices

1. **Always use guards on flows with generic keywords** (hello, hi, help, menu)
2. **Check state before clearing it** — verify no active flow first
3. **Use specific keywords when possible** — "book appointment" instead of "hello"
4. **Document your keyword strategy** — list all flows and their triggers
5. **Test mid-conversation interruptions** — what happens if user types "hello" at step 3 of a 5-step flow?

---

### Guard Pattern Template

Use this template for any flow that might collide with others:

```typescript
const myFlow = addKeyword(['keyword1', 'keyword2'])
    .addAction(async (ctx, { state }) => {
        // ✅ GUARD: Check if user is in another flow
        const activeIndicators = [
            await state.get('clientName'),
            await state.get('activeBooking'),
            await state.get('slotsCache'),
            // ... add your flow-specific state keys
        ]
        
        if (activeIndicators.some(Boolean)) {
            console.log('[GUARD] User in active flow, skipping trigger')
            return  // Exit early
        }
        
        // Optionally: mark this flow as active
        await state.update({ activeFlow: 'myFlow' })
    })
    .addAnswer('Start of flow...')
    // ... rest of flow ...
    .addAction(async (ctx, { state, endFlow }) => {
        // ✅ Clear active flag when done
        await state.update({ activeFlow: null })
        return endFlow()
    })
```

---

### Summary

**Rule**: Flows with overlapping keywords will trigger simultaneously unless you add guards.

**Why**: BuilderBot evaluates all flows against every message. No automatic exclusion.

**Solutions**:
1. **Guard with state check** — verify user isn't mid-flow before triggering
2. **Use `activeFlow` flag** — explicit flow state management
3. **Design non-overlapping keywords** — minimize collisions

**Best practice**: Always guard flows with generic keywords (hello, hi, help, menu) by checking state first.

---

**Previous**: [Gotcha #2: gotoFlow() + capture Bug](#gotcha-2-gotoflow--capture-bug)  
**Next**: [Gotcha #4: State Persistence Gotcha](#) (coming soon)
