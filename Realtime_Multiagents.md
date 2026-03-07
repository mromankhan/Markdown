Now I will tell you the **straight truth — after reading the docs.**

---

## Realtime Multi-Agent is ABSOLUTELY Possible

Your instinct is **100% correct**.

Confirmed from the docs:

```python
# LiveKit docs example - this works in REALTIME as well
@function_tool()
async def transfer_to_billing(self, context: RunContext):
    """Transfer to billing specialist."""
    return BillingAgent(chat_ctx=self.chat_ctx), "Transferring to billing"
```

LiveKit’s handoff mechanism — **returning an Agent instance from `function_tool`** — works for both **pipeline and realtime**.

There is **no restriction in the docs**.

---

## How Realtime Multi-Agent Works (Internally)

```
Pipeline Multi-Agent:

Parent LLM (openai.LLM) → handoff → Subagent LLM (openai.LLM)

[DIFFERENT HTTP clients, different models possible]


Realtime Multi-Agent:

Parent RealtimeModel (1 WebSocket) → handoff → Subagent

[SAME WebSocket connection, LiveKit calls update_instructions internally]
```

When a handoff happens in realtime:

1. LiveKit `AgentSession` receives a new Agent instance
2. The **same RealtimeModel WebSocket remains** — it is not closed or reopened
3. The session instructions are updated with the sub-agent’s `sysMsg`
4. `on_enter()` is called and the context is merged
5. The sub-agent **cannot use a different voice** (same RealtimeModel = same voice) — this is a limitation

This is also explicitly mentioned in the docs:

Agents can override **`tts`, `llm`, `stt` at the agent level** — if you pass a different `RealtimeModel` in the sub-agent constructor, it will use a **different WebSocket connection**.

---

## In Our Codebase There Are Only 3 Actual Blockers

### Blocker 1 — Explicit Guard (`factory.py:120-122`)

```python
if is_multi_agent:
    logger.warning("⚠️ Multi-agent not supported for realtime architecture, using single-agent mode")
    # subagents silently ignored
```

**Fix:**
Simply remove this check or create a separate multi-agent path for realtime.

---

### Blocker 2 — UserData Dict vs Dataclass (`factory.py:147-155`)

```python
# For realtime this currently happens:
session.userdata = {          # plain dict
    'kb_settings': ...,
    # personas? not present
    # can_handoff()? not present
}
```

The handoff tool does this:

```python
userdata = ctx.userdata
userdata.can_handoff()           # 💥 AttributeError — dict has no method
userdata.personas.get(target_id) # 💥 KeyError — 'personas' not in dict
```

**Fix:**
Use the `UserData` dataclass for realtime multi-agent as well.

---

### Blocker 3 — No LLM is Created for Subagents in the Realtime Path

Currently in realtime only **one `realtime_llm`** is created.

For subagents you must either:

* Share the **same `realtime_llm`** (same voice, same model — simpler)
* Or create a **separate `RealtimeModel` for each subagent** (different voice possible — more complex)

---

## Implementation Plan (If You Want to Implement It)

### Approach A — Shared RealtimeModel (Simpler, Same Voice)

Realtime multi-agent path in `factory.py`:

```python
if agent_config.agent_architecture == "realtime":
    if is_multi_agent and agent_config.subagents:

        # Create UserData dataclass — not dict
        userdata = UserData(
            ctx=ctx,
            agent_config=agent_config,
            kb_settings=...,
            ...
        )

        # Parent agent — will use shared realtime_llm
        parent_agent = UnifiedAgent(
            agent_config=agent_config,
            userdata=userdata,
            llm=realtime_llm,  # shared
            ...
        )

        userdata.personas["parent"] = parent_agent

        # Subagents — pass SAME realtime_llm
        for sub_config in agent_config.subagents:
            subagent = UnifiedAgent(
                agent_config=sub_config,
                userdata=userdata,
                llm=realtime_llm,  # same WebSocket connection
                ...
            )

            userdata.personas[sub_config.id] = subagent

        # Session with typed UserData
        session = AgentSession[UserData](
            llm=realtime_llm,
            vad=vad_instance,
            tts=tts_instance,
            userdata=userdata,
        )

        return session, userdata, parent_agent
```

---

### Approach B — Separate RealtimeModel per Subagent (Different Voice per Agent)

```python
# Separate realtime model for parent
parent_realtime_llm = initialize_realtime_llm(...)

# Separate realtime model for each subagent
subagent_realtime_llm = initialize_realtime_llm(
    realtime_model=sub_config.realtime_provider or agent_config.realtime_provider,
    instructions=sub_config.sysMsg,
    ...
)

subagent = UnifiedAgent(llm=subagent_realtime_llm, ...)
```

This creates **separate WebSocket connections** — it is possible but uses more resources.

---

## Summary

|                   | Pipeline                | Realtime (Current)  | Realtime (Possible)                    |
| ----------------- | ----------------------- | ------------------- | -------------------------------------- |
| Multi-agent       | ✅                       | ❌ (blocked by code) | ✅ implementable                        |
| Handoff mechanism | `return Agent`          | same would work     | same would work                        |
| UserData          | dataclass               | dict ❌              | dataclass required                     |
| Subagent voice    | different TTS per agent | same RealtimeModel  | same (A) or separate RealtimeModel (B) |
| WebSocket         | N/A                     | 1 connection        | 1 shared (A) or N connections (B)      |

---

## Should It Be Implemented?

**Approach A (shared RealtimeModel)** — **relatively simple, implement it.**

Handoff will work, only the **voice will remain the same for all agents**.

For production this is usually enough:

Different **persona = different instructions**, same voice.

---

**Approach B** — requires more engineering, and **OpenAI / Google realtime APIs will also cost more** (multiple WebSocket sessions).

Only use this if **voice differentiation is critical**.

---

**What do you want to implement?**

I can update `factory.py`.
