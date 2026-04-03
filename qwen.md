### Question: I want to use Qwen’s realtime omni models how can I do that?


**Short answer: Not possible directly.**

The issue with Qwen-Omni is:

| Feature   | Qwen-Omni (DashScope)  | LiveKit Realtime Requirement |
| --------- | ---------------------- | ---------------------------- |
| Protocol  | HTTP SSE (stream=True) | **Bidirectional WebSocket**  |
| Direction | Request → Response     | Full duplex (simultaneous)   |

LiveKit’s realtime architecture (OpenAI Realtime, Gemini Live, Ultravox) is all WebSocket-based — audio flows continuously in both directions. Qwen-Omni only supports HTTP streaming, which is fundamentally different.

**What can practically work:**

1. **Pipeline (already done)** — Qwen as LLM in STT → LLM → TTS — this is the best option and is working
2. **Custom RealtimeModel plugin** — Wrap DashScope HTTP streaming into LiveKit’s RealtimeModel interface — very complex, and latency will be higher, not suitable for production

**Recommendation:** Use the pipeline architecture for Qwen. For realtime, `fixie-ai/ultravox-llama3.3-70b` (which we just set up) is a solid model — there is currently no available Ultravox + Qwen backbone.





--------------------------------



### Question: is it really not possible to add any qwen realtime model?

**Confirm: Not possible.** Research complete.

**Root cause:**

No model from DashScope/Qwen supports bidirectional WebSocket audio streaming. It only uses HTTP SSE (stream=True) — which is not compatible with LiveKit’s `RealtimeModel` interface.

The realtime plugins that exist in LiveKit (OpenAI, Gemini, Ultravox, xAI) all use **WebSocket full-duplex**. Ultravox had tried a Qwen backbone (`qwen3-32b-preview`), but it was deprecated and no replacement has been released.

**Only possible paths for Qwen realtime:**

1. Alibaba releases its own WebSocket realtime API (currently not available)
2. Ultravox adds a new Qwen backbone (under their control)

**Your best options right now:**

* **Realtime:** `fixie-ai/ultravox-llama3.3-70b` (which we just set up)
* **Qwen specifically:** Pipeline architecture — `qwen-max` / `qwen-plus` via DashScope using STT → LLM → TTS

The pipeline setup is already working — restart and test it.
