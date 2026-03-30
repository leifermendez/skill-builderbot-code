# BuilderBot Production Gotchas

> Critical bugs discovered in real production implementations that don't show error messages but silently break flows.

This document consolidates hard-won lessons from BuilderBot bots running in production, handling real users and real conversations. These gotchas are **silent killers** — they don't throw errors, they just make your bot stop responding, leaving you wondering what went wrong.

**Source**: Patterns extracted from production medical appointment booking systems, e-commerce bots, and customer support implementations.

---

## ⚠️ Gotcha #1: Flow Terminal Bug

### The Problem

When the **last** `addAnswer` in a flow chain has `capture: true`, and it only updates state without calling any flow-control method (`gotoFlow`, `endFlow`, `fallBack`), the bot becomes **completely silent** after the user responds.

**Symptoms:**
- Bot stops responding
- No error in console
- No timeout warning
- User is stuck waiting
- Flow appears to end successfully but nothing happens

This is one of the most frustrating bugs because there's **no visible error** — the flow just dies silently.

---

### Why It Happens

BuilderBot expects terminal flows (the last step in a conversation chain) to be **self-contained** — meaning they should:
1. Capture the user's input
2. Process it (e.g., create appointment, save to DB, call API)
3. Send confirmation message (`flowDynamic`)
4. Clean up and close (`endFlow` or navigate to another flow)

If step 3 or 4 is missing, BuilderBot completes the callback successfully but has nowhere to go next. The conversation context remains "active" but frozen — no new messages trigger any response.

**Root cause**: The last capture point in a flow chain must handle the full lifecycle inline. It cannot delegate to another flow via `gotoFlow` because there's nothing after it to receive control.

---

### Wrong vs Right

#### ❌ Wrong — Silent Death

```typescript
// customDate.flow.ts
const customDateFlow = addKeyword('custom_date')
    .addAnswer('What date works for you?', { capture: true }, async (ctx, { state }) => {
        await state.update({ selectedDate: ctx.body })
        // 💀 Flow ends here silently — no confirmation, no navigation, nothing
    })

export default customDateFlow
```

**What happens:**
1. User types "March 25"
2. State updates with `selectedDate: "March 25"`
3. Callback completes successfully ✅
4. Bot goes silent 🤐 — no confirmation, no next step

**User experience**: "Is the bot broken? Did it save my date?"

---

#### ✅ Right — Self-Contained Terminal Flow

```typescript
// customDate.flow.ts
const customDateFlow = addKeyword('custom_date')
    .addAnswer('What date works for you?', { capture: true }, async (ctx, { state, flowDynamic, endFlow }) => {
        const userDate = ctx.body
        
        // 1. Validate input
        if (!isValidDate(userDate)) {
            return fallBack('That doesn't look like a valid date. Please try again (e.g., "March 25"):')
        }
        
        // 2. Update state
        await state.update({ selectedDate: userDate })
        
        // 3. Process (e.g., create appointment)
        const appointment = await createAppointment({
            date: userDate,
            clientName: state.get('clientName'),
            // ... other fields from state
        })
        
        // 4. Send confirmation
        await flowDynamic([
            {
                body: `✅ Perfect! Your appointment is confirmed for ${userDate}.`,
                delay: Math.floor(Math.random() * 800) + 500
            }
        ])
        
        // 5. Clean up and close
        state.clear()
        return endFlow('See you then! 😊')
    })

export default customDateFlow
```

**What happens:**
1. User types "March 25"
2. Input is validated
3. State updates
4. Appointment is created
5. User receives confirmation ✅
6. Flow ends gracefully ✅

**User experience**: "Great! My appointment is confirmed."

---

### Alternative Pattern — Delegate Before Terminal Capture

If you need to split logic across flows, capture all required data **before** the terminal flow:

```typescript
// mainMenu.flow.ts
const mainMenuFlow = addKeyword(['hello', 'hi'])
    .addAnswer('What's your name?', { capture: true }, async (ctx, { state }) => {
        await state.update({ clientName: ctx.body })
    })
    .addAnswer('What date works for you?', { capture: true }, async (ctx, { state, gotoFlow }) => {
        await state.update({ selectedDate: ctx.body })
        
        // ✅ Now we can delegate because this is NOT the last capture
        const { confirmationFlow } = await import('./confirmation.flow')
        return gotoFlow(confirmationFlow)
    })

// confirmation.flow.ts
const confirmationFlow = addKeyword('CONFIRMATION_INTERNAL')
    .addAction(async (ctx, { state, flowDynamic, endFlow }) => {
        const { clientName, selectedDate } = state.getMyState()
        
        // Create appointment
        await createAppointment({ clientName, selectedDate })
        
        // Confirm and close
        await flowDynamic(`✅ Confirmed for ${selectedDate}, ${clientName}!`)
        state.clear()
        return endFlow()
    })
```

**Key difference**: The terminal action is in `confirmationFlow.addAction`, not in a capture. This allows proper delegation.

---

### How to Debug

If your bot goes silent after a capture:

1. **Check the last `addAnswer` with `capture: true`** in your flow chain
2. **Verify it does ALL of these**:
   - ✅ Sends a message to the user (`flowDynamic` or returns `endFlow('message')`)
   - ✅ Calls a flow-control method (`return gotoFlow(...)` or `return endFlow(...)`)
   - ✅ OR is followed by another `addAnswer`/`addAction` (not terminal)

3. **Add debug logs** to confirm the callback is executing:
   ```typescript
   .addAnswer('...', { capture: true }, async (ctx, { state }) => {
       console.log('[DEBUG] Last capture executed, body:', ctx.body)
       await state.update({ data: ctx.body })
       console.log('[DEBUG] State updated, but now what?') // 💀 This is where it dies
   })
   ```

4. **The fix**: Make it self-contained by adding `flowDynamic` + `endFlow` inline

---

### Real-World Example

**Scenario**: Medical appointment booking bot (AnitaChatBot-Odontologia)

**Bug**: After implementing a custom date search flow, users could search for dates but the bot went silent after they selected one. The appointment was created in the backend (logs confirmed it), but users never received confirmation.

**Root cause**: The last capture in `customDate.flow.ts` was:
```typescript
.addAnswer('Which slot?', { capture: true }, async (ctx, { state }) => {
    await state.update({ selectedSlot: ctx.body })
    // 💀 Ended here — appointment created but no confirmation sent
})
```

**Fix**: Moved appointment creation and confirmation into the same callback:
```typescript
.addAnswer('Which slot?', { capture: true }, async (ctx, { state, flowDynamic, endFlow }) => {
    const slot = ctx.body
    const appointment = await createAppointment({ slot, ...state.getMyState() })
    
    await flowDynamic(`✅ Confirmed! Your appointment is on ${appointment.date} at ${appointment.time}.`)
    state.clear()
    return endFlow('See you soon! 😊')
})
```

**Result**: Users now receive confirmation every time. Bug fixed.

---

### Summary

**Rule**: The last `capture: true` in a flow chain must be **self-contained** — it must send a message and close/navigate inline. It cannot delegate to another flow.

**Why**: BuilderBot has nowhere to go after the terminal capture completes. If you don't explicitly tell it what to do next, it goes silent.

**Solution**: Either make the terminal capture self-contained, or ensure it's not actually terminal (followed by more steps).

---

**Next**: [Gotcha #2: gotoFlow() + capture Bug](#) (coming soon)
