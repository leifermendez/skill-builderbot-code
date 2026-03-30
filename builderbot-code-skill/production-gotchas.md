# BuilderBot Production Gotchas

> Critical bugs discovered in real production implementations that don't show error messages but silently break flows.

This document consolidates hard-won lessons from BuilderBot bots running in production, handling real users and real conversations. These gotchas are **silent killers** — they don't throw errors, they just make your bot stop responding, leaving you wondering what went wrong.

**Source**: Patterns extracted from production medical appointment booking systems, e-commerce bots, and customer support implementations.

---

## ⚠️ Gotcha #4: State Persistence Gotcha

### The Problem

BuilderBot's `state` is **volatile** — it's stored in a JavaScript `Map` in memory and is **completely lost** when:

**Triggers for state loss:**
- Bot restarts (manual restart, code deploy, server reboot)
- Process crashes (unhandled exception, out of memory, killed by OS)
- PM2 restart (automatic or manual)
- Docker container restart
- Server restart or update

**Symptoms:**
- User is mid-conversation when bot restarts
- After restart, state is empty
- Bot treats returning user as new user
- Multi-step flows break (user was on step 3, now state is gone)
- No way to resume conversation where it left off

This is **by design** — `state` is not meant to be persistent. But many developers assume it persists, leading to data loss in production.

---

### Why It Happens

BuilderBot's `state` implementation is intentionally simple:

```typescript
// Simplified internal implementation (conceptual)
class StateManager {
    private store = new Map<string, any>()  // ← In-memory only
    
    async update(data: any) {
        this.store.set(userId, data)  // ← Lost on restart
    }
}
```

**Why it's volatile:**
- **Performance** — no I/O overhead (disk writes, DB queries)
- **Simplicity** — no serialization, no external dependencies
- **Intended use case** — short-lived conversational state (name, current step, temp data)

**Not a bug, it's a feature** — but it catches developers off guard.

---

### Wrong vs Right

#### ❌ Wrong — Trusting State to Persist Critical Data

```typescript
const bookingFlow = addKeyword(['book'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAnswer('What date?', { capture: true }, async (ctx, { state }) => {
        await state.update({ appointmentDate: ctx.body })
    })
    .addAnswer('Confirm? (yes/no)', { capture: true }, async (ctx, { state }) => {
        if (ctx.body.toLowerCase() === 'yes') {
            const { clientName, appointmentDate } = state.getMyState()
            
            // ❌ Problem: if bot restarts between "date" and "confirm",
            // state is lost and this will fail
            await createAppointment({ clientName, appointmentDate })
        }
    })
```

**Scenario:**
1. User starts booking: "book"
2. Bot: "What's your name?"
3. User: "Jorge"
4. Bot: "What date?"
5. User: "March 25"
6. **🔄 Bot restarts (deploy, crash, PM2 restart)** ← State wiped
7. User: "yes" (to confirm)
8. 💀 `state.getMyState()` returns empty object
9. 💀 Appointment creation fails or creates invalid appointment
10. 💀 User thinks their appointment is confirmed, but it's not

**User experience**: "I confirmed my appointment but it's not showing up. This bot is broken!"

---

#### ✅ Right — Sync State to External Storage for Critical Data

**Solution 1: Write to database/API after each critical step**

```typescript
const bookingFlow = addKeyword(['book'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
        
        // ✅ Also write to external storage immediately
        await savePartialBooking(ctx.from, { clientName: ctx.body })
    })
    .addAnswer('What date?', { capture: true }, async (ctx, { state }) => {
        await state.update({ appointmentDate: ctx.body })
        
        // ✅ Update external storage
        await updatePartialBooking(ctx.from, { appointmentDate: ctx.body })
    })
    .addAnswer('Confirm? (yes/no)', { capture: true }, async (ctx, { state }) => {
        if (ctx.body.toLowerCase() === 'yes') {
            // ✅ Read from external storage (fallback if state is lost)
            const booking = await getPartialBooking(ctx.from) || state.getMyState()
            
            if (!booking.clientName || !booking.appointmentDate) {
                return fallBack('Sorry, I lost your info. Let's start over. What's your name?')
            }
            
            await createAppointment(booking)
            
            // ✅ Clean up external storage
            await deletePartialBooking(ctx.from)
            state.clear()
        }
    })
```

**Key principle**: State is a **cache**, external storage is the **source of truth**. Write critical data to both.

---

**Solution 2: Use database state from the beginning (not volatile state)**

```typescript
// Use MongoDB adapter's state instead of BuilderBot's volatile state
const bookingFlow = addKeyword(['book'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { database }) => {
        // ✅ Write directly to MongoDB (persists across restarts)
        await database.save({
            userId: ctx.from,
            data: { clientName: ctx.body, step: 'name' }
        })
    })
    .addAnswer('What date?', { capture: true }, async (ctx, { database }) => {
        const user = await database.get(ctx.from)
        
        await database.save({
            userId: ctx.from,
            data: { ...user.data, appointmentDate: ctx.body, step: 'date' }
        })
    })
    .addAnswer('Confirm? (yes/no)', { capture: true }, async (ctx, { database }) => {
        const user = await database.get(ctx.from)
        
        if (ctx.body.toLowerCase() === 'yes') {
            await createAppointment(user.data)
            await database.delete(ctx.from)  // Clean up
        }
    })
```

**Key principle**: Skip volatile `state` entirely for critical flows. Use the database adapter directly.

---

**Solution 3: Detect state loss and recover gracefully**

```typescript
const bookingFlow = addKeyword(['book'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body, startedAt: Date.now() })
    })
    .addAnswer('What date?', { capture: true }, async (ctx, { state }) => {
        await state.update({ appointmentDate: ctx.body })
    })
    .addAnswer('Confirm? (yes/no)', { capture: true }, async (ctx, { state, fallBack }) => {
        const stateData = state.getMyState()
        
        // ✅ Detect state loss
        if (!stateData.clientName || !stateData.appointmentDate) {
            console.log('[STATE LOSS] Detected empty state at confirmation step')
            
            // Option A: Ask user to start over
            state.clear()
            return fallBack('Sorry, I lost your booking info (bot restarted). Let's start fresh. What's your name?')
            
            // Option B: Try to recover from external storage
            // const recovered = await tryRecoverBooking(ctx.from)
            // if (recovered) { ... }
        }
        
        if (ctx.body.toLowerCase() === 'yes') {
            await createAppointment(stateData)
            state.clear()
        }
    })
```

**Key principle**: Always validate that state contains expected data before using it. Handle empty state gracefully.

---

### How to Debug

If users report "lost bookings" or "bot forgot my info":

1. **Check if bot restarted during user's session**:
   ```bash
   # Check PM2 restart logs
   pm2 logs --lines 100 | grep -i restart
   
   # Check when process started
   pm2 list
   ```

2. **Add state validation at critical points**:
   ```typescript
   .addAction(async (ctx, { state }) => {
       const stateData = state.getMyState()
       console.log('[STATE CHECK]', {
           userId: ctx.from,
           hasName: !!stateData.clientName,
           hasDate: !!stateData.appointmentDate,
           keys: Object.keys(stateData)
       })
   })
   ```

3. **Monitor bot uptime and correlate with user complaints**:
   ```typescript
   const startTime = Date.now()
   
   // In your health check endpoint:
   app.get('/health', (req, res) => {
       res.json({
           uptime: Date.now() - startTime,
           uptime_minutes: Math.floor((Date.now() - startTime) / 60000)
       })
   })
   ```

4. **If state loss is confirmed** → implement one of the three solutions above

---

### Real-World Example

**Scenario**: Medical appointment booking bot deployed with PM2

**Bug**: Users would report "I confirmed my appointment but it's not in the system." Investigation showed:
1. PM2 was set to auto-restart every 24 hours (memory leak mitigation)
2. Restarts happened at random times during the day
3. Users mid-conversation during restart would lose all state
4. Confirmation step would fail silently (no name/date in state)

**Root cause**: Trusting volatile state for critical appointment data

**Timeline of a lost booking**:
```
10:45 AM - User starts booking
10:46 AM - Provides name "Jorge"
10:47 AM - Provides date "March 25"
10:48 AM - PM2 restarts bot (scheduled 24h restart) ← STATE WIPED
10:49 AM - User types "yes" to confirm
10:49 AM - Bot tries to create appointment with empty state → fails
10:49 AM - User sees "Confirmed!" message (lie)
10:50 AM - User checks system → no appointment
```

**Fix applied**: Implemented Solution #1 (sync to MongoDB after each step)
```typescript
// After each capture, write to external storage
.addAnswer('What's your name?', { capture: true }, async (ctx, { state, database }) => {
    await state.update({ clientName: ctx.body })
    
    // ✅ CRITICAL: Also persist to MongoDB
    await database.save({
        userId: ctx.from,
        data: { clientName: ctx.body, step: 'name', timestamp: Date.now() }
    })
})

// At confirmation, read from MongoDB (survives restarts)
.addAnswer('Confirm?', { capture: true }, async (ctx, { state, database }) => {
    if (ctx.body.toLowerCase() === 'yes') {
        // ✅ Try state first (fast), fall back to DB if empty
        let booking = state.getMyState()
        
        if (!booking.clientName) {
            console.log('[RECOVERY] State empty, reading from DB')
            const dbUser = await database.get(ctx.from)
            booking = dbUser?.data || {}
        }
        
        if (!booking.clientName || !booking.appointmentDate) {
            return fallBack('Sorry, please start over. What's your name?')
        }
        
        await createAppointment(booking)
        await database.delete(ctx.from)  // Cleanup
        state.clear()
    }
})
```

**Result**: Zero lost bookings after fix. Bot now survives restarts gracefully.

---

### Best Practices

1. **Treat state as a cache, not as persistent storage**
2. **Write critical data to external storage (DB, API) immediately**
3. **Always validate state before using it** — check for expected keys
4. **Design flows to handle state loss gracefully** — ask user to start over, or recover from external storage
5. **Monitor bot restarts and correlate with user complaints** — use uptime metrics
6. **For critical multi-step flows (payments, bookings)** → sync to DB after every step
7. **Document state volatility in your team docs** — make it explicit

---

### When to Use Volatile State vs External Storage

| Use Case | Volatile State ✅ | External Storage ✅ |
|----------|------------------|---------------------|
| Current step in flow | Yes | Optional |
| User's name (just collected) | Yes | If used later |
| Temp data (parsed intent, current menu) | Yes | No |
| Appointment booking in progress | **Sync to both** | **Primary** |
| Payment info | **Never** | **Always** |
| User preferences (language, timezone) | No | **Always** |
| Session-only flags (showedWelcome) | Yes | No |

**Rule of thumb**: If losing the data would frustrate the user or cause business impact, write it to external storage.

---

### State Persistence Pattern Template

Use this template for critical multi-step flows:

```typescript
const criticalFlow = addKeyword(['start'])
    .addAnswer('Step 1?', { capture: true }, async (ctx, { state, database }) => {
        const data = { step1: ctx.body }
        
        // Write to both (state is cache, DB is source of truth)
        await state.update(data)
        await database.save({ userId: ctx.from, data, timestamp: Date.now() })
    })
    .addAnswer('Step 2?', { capture: true }, async (ctx, { state, database }) => {
        // Read from state (fast), update DB
        const current = state.getMyState()
        const updated = { ...current, step2: ctx.body }
        
        await state.update(updated)
        await database.save({ userId: ctx.from, data: updated, timestamp: Date.now() })
    })
    .addAnswer('Confirm?', { capture: true }, async (ctx, { state, database, fallBack }) => {
        if (ctx.body.toLowerCase() === 'yes') {
            // Try state first, fall back to DB if empty (restart recovery)
            let data = state.getMyState()
            
            if (!data.step1 || !data.step2) {
                console.log('[RECOVERY] State empty, reading from DB')
                const dbUser = await database.get(ctx.from)
                data = dbUser?.data || {}
            }
            
            // Validate we have required data
            if (!data.step1 || !data.step2) {
                return fallBack('Lost your data. Start over: ...')
            }
            
            // Process with recovered/cached data
            await processCriticalAction(data)
            
            // Clean up both
            await database.delete(ctx.from)
            state.clear()
        }
    })
```

---

### Summary

**Rule**: BuilderBot's `state` is volatile — it's lost on every bot restart.

**Why**: State is an in-memory `Map` designed for performance, not persistence.

**Solutions**:
1. **Sync critical data to external storage** after each step (DB, API)
2. **Use database adapter directly** for critical flows (skip volatile state)
3. **Detect state loss and recover** gracefully (validate + fallback or recover from DB)

**Best practice**: Treat state as a cache. For critical data (bookings, payments), always sync to external storage immediately.

---

**Previous**: [Gotcha #3: Guard Pattern for Duplicate Flows](#gotcha-3-guard-pattern-for-duplicate-flows)  
**End of Production Gotchas series** ✅

---

## 🎓 Conclusion

These four gotchas are responsible for the majority of "silent failures" in production BuilderBot implementations:

1. **Flow Terminal Bug** — last capture must be self-contained
2. **gotoFlow() + capture Bug** — race condition in handler registration
3. **Guard Pattern** — overlapping keywords trigger multiple flows
4. **State Persistence** — volatile state lost on restart

**All discovered and solved in real production bots** handling medical appointments, customer support, and e-commerce transactions.

**Key takeaway**: These bugs don't throw errors. They just make your bot stop responding or lose user data. That's what makes them so dangerous.

If you've discovered other production gotchas, please contribute! Open an issue or PR at [leifermendez/skill-builderbot-code](https://github.com/leifermendez/skill-builderbot-code).
