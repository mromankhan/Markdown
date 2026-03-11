# Agent Break Detection — Test Implementation Plan

## Overview

Three test files to create + one small code change in `entrypoint.py`.

```
tests/
├── test_cleanup_integrity.py       ← Option A (new)
├── test_background_task_safety.py  ← Option B (new)
├── test_provider_smoke.py          ← Option C (new)
entrypoint.py                       ← small refactor needed for Option A
```

---

## Option A — Cleanup Integrity Tests

### Problem
`protected_cleanup` is a nested function inside `entrypoint()` — it cannot be imported directly for testing.

### Code Change Required in `entrypoint.py`

Extract cleanup steps into a module-level async function `run_cleanup_sequence()`:

```python
# NEW function at module level (not nested)
async def run_cleanup_sequence(
    room_name: str,
    billing_session_started: bool,
    billing_session_info: dict,
    agent_config,
    transcription_segments: list,
    turn_latencies: list,
    room_type: str,
    user_id: str,
    user_identifier: str,
    session_start_time,
):
    # Step 0: stop_recording (background)
    # Step 1: end_billing + consume_credits
    # Step 2: parallel — transcription + analysis + graphiti
    # Step 5: delete_room
```

`protected_cleanup` inside `entrypoint()` just calls `await run_cleanup_sequence(...)` passing all locals.

### Test File: `tests/test_cleanup_integrity.py`

| # | Test Name | Input | Assert | Detects |
|---|---|---|---|---|
| 1 | `test_all_steps_execute_on_normal_end` | normal session, billing_session_started=True, 3 segments | end_billing ✅ save_transcription ✅ trigger_analysis ✅ delete_room ✅ | Any step silently removed from cleanup |
| 2 | `test_cleanup_continues_when_billing_fails` | end_billing_session raises Exception | save_transcription still called ✅, delete_room still called ✅ | Billing exception stopping the rest of cleanup |
| 3 | `test_analysis_skipped_when_no_transcription` | transcription_segments = [] | save_transcription NOT called, trigger_analysis NOT called, delete_room ✅ | Analysis running on empty call or room not deleted |
| 4 | `test_graphiti_timeout_does_not_block_transcription` | ingest_conversation_async raises asyncio.TimeoutError | save_transcription called ✅ (must not wait for graphiti), delete_room ✅ | Graphiti timeout hanging entire cleanup |
| 5 | `test_delete_room_always_called_even_if_everything_fails` | billing fails, transcription fails, graphiti fails, analysis fails | delete_room called ✅ | Orphaned LiveKit rooms when all services are down |
| 6 | `test_billing_skipped_when_session_never_started` | billing_session_started=False | end_billing NOT called, consume_credits NOT called, rest runs ✅ | Billing called when session was never started |

---

## Option B — Background Task Safety Tests

### Problem
`entrypoint.py` lines 522–524:
```python
asyncio.create_task(stop_recording(room_name))       # no error handling
asyncio.create_task(merge_recording_via_api(room_name))  # no error handling
```
If these crash, nobody knows.

### Code Change Required in `entrypoint.py`

Add a `_make_task_error_handler` function and attach it to every `create_task()` call:

```python
def _make_task_error_handler(task_name: str):
    """Returns a done_callback that logs exceptions from fire-and-forget tasks."""
    def _handler(task: asyncio.Task):
        if task.cancelled():
            return
        exc = task.exception()
        if exc:
            logger.error(
                f"❌ Background task '{task_name}' raised unhandled exception: {exc}",
                exc_info=exc
            )
    return _handler

# Usage everywhere create_task is called:
task = asyncio.create_task(stop_recording(room_name))
task.add_done_callback(_make_task_error_handler("stop_recording"))
```

Tasks that need the handler attached:
- `stop_recording`
- `merge_recording_via_api`
- `background_setup`
- `agent_say` (greeting + inactivity)

### Test File: `tests/test_background_task_safety.py`

| # | Test Name | Input | Assert | Detects |
|---|---|---|---|---|
| 1 | `test_handler_catches_exception_from_failing_task` | task raises RuntimeError("Recording API down") | done_callback fires, exception captured | Handler function itself broken |
| 2 | `test_handler_does_not_fire_for_successful_task` | task completes normally | handler fires, task.exception() is None | False positives in error handler |
| 3 | `test_handler_does_not_fire_for_cancelled_task` | task gets cancelled | handler fires, task.cancelled() is True, no error logged | False positives on intentional cancellation |
| 4 | `test_stop_recording_failure_logged_with_task_name` | mock stop_recording raises Exception | error log contains "stop_recording" | Task name missing from logs |
| 5 | `test_multiple_background_task_failures_all_logged` | stop_recording fails + merge_recording fails | both errors logged separately | One failure masking another |
| 6 | `test_handler_attached_to_all_background_tasks` | inspect entrypoint.py / mock create_task | stop_recording, merge_recording, background_setup, agent_say all have done_callback | Developer adding new create_task without handler |

---

## Option C — Provider Smoke Tests

### No Code Change Required

Directly calls real service initialization functions with real env keys.

### Test File: `tests/test_provider_smoke.py`

| # | Test Name | Input | Assert | Requires | Detects |
|---|---|---|---|---|---|
| 1 | `test_billing_service_reachable` | POST /billing/credits/check | 200 or 402 (not 500, not connection error) | Running FastAPI server | Billing service down or wrong port |
| 2 | `test_llm_init_google` | initialize_llm("gemini-2.0-flash-001", 0.7, ...) | Returns LLM instance, no exception | GOOGLE_API_KEY | Breaking change in LLM provider init |
| 3 | `test_llm_init_openai` | initialize_llm("gpt-4o-mini", 0.7, ...) | Returns LLM instance, no exception | OPENAI_API_KEY | OpenAI key invalid, SDK version mismatch |
| 4 | `test_unknown_model_falls_back_to_openai` | initialize_llm("some-unknown-model-xyz", ...) | Returns LLM instance (OpenAI fallback), warning logged | Any one API key | Unknown model crashing instead of falling back |
| 5 | `test_agent_config_loads_from_db` | load_agent_configuration(room_name_with_valid_agent) | Returns UnifiedAgentConfig, not None, required fields present | DATABASE_URI + TEST_AGENT_ID env var | DB schema change breaking agent loading |
| 6 | `test_room_name_parser_handles_all_types` | 8 room name formats: voice_*, inbound_*, outbound_*, demo_*, chat_*, agent_*, mock_*, malformed | parse_room_name() returns dict with 'type' key for all inputs | None | Room name format change breaking the parser |
| 7 | `test_check_credits_fail_closed_on_500` | billing service mocked to return 500 | check_credits() raises Exception (NOT returns False or None) | None | Fail-closed behavior accidentally changed to fail-open |

---

## Execution Strategy

### Which tests need API keys

```
test_cleanup_integrity.py      → No keys needed (all mocked)
test_background_task_safety.py → No keys needed (pure async logic)
test_provider_smoke.py:
  Test 1 (billing reachable)   → Needs running FastAPI server
  Test 2,3 (LLM init)          → Needs GOOGLE_API_KEY / OPENAI_API_KEY
  Test 4 (fallback)            → Needs any one API key
  Test 5 (agent config)        → Needs DATABASE_URI + TEST_AGENT_ID
  Test 6 (room parser)         → No keys needed
  Test 7 (fail-closed)         → No keys needed (billing mocked)
```

### How to run

```bash
# Fast tests — no API keys needed
pytest tests/test_cleanup_integrity.py tests/test_background_task_safety.py -v

# Smoke tests — needs env vars
pytest tests/test_provider_smoke.py -v -m smoke

# All three together
pytest tests/test_cleanup_integrity.py tests/test_background_task_safety.py tests/test_provider_smoke.py -v
```

### Pytest marks to add in `conftest.py`

```python
config.addinivalue_line("markers", "smoke: Provider availability tests (require API keys)")
config.addinivalue_line("markers", "integrity: Cleanup sequence integrity tests")
config.addinivalue_line("markers", "task_safety: Background task error handling tests")
```

---

## Summary

| Option | Files Changed | Tests Added | Keys Needed | What It Detects |
|---|---|---|---|---|
| **A — Cleanup Integrity** | `entrypoint.py` (extract `run_cleanup_sequence`) + new test file | 6 tests | None | Any cleanup step silently skipped or broken |
| **B — Background Task Safety** | `entrypoint.py` (add `_make_task_error_handler`, attach to 4 tasks) + new test file | 6 tests | None | Silent background task failures |
| **C — Provider Smoke** | No prod code change + new test file | 7 tests | Partial (3 of 7 tests) | Provider/DB/config breaks at startup |
| **Total** | 1 file changed, 3 new test files | 19 tests | — | ~55-60% of all failure surface |

---

## Coverage Gap Reminder

These 3 options cover **~55-60%** of failure surface. Remaining gaps:

| Gap | Solution | Not Covered By Tests |
|---|---|---|
| Worker process crash mid-session | Heartbeat monitoring (Redis) | All 3 options |
| Full STT→LLM→TTS voice pipeline | LiveKit E2E test (real room) | All 3 options |
| Multi-agent handoff failure | Dedicated handoff tests | All 3 options |
| Webhook delivery failure | Webhook retry queue in DB | Option C partially |
| Race conditions in billing | Load tests | All 3 options |
