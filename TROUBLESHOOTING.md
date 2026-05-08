# Troubleshooting: Telegram Typing Indicator Disappears During Long Requests

**Date:** 2026-05-08
**Context:** After fixing NKQ curl_cffi patch (real SSE streaming), Telegram typing indicator disappears during long requests.

---

## Symptoms

- Telegram shows typing indicator for ~5 seconds, then disappears.
- Indicator reappears when Hermes sends a final message.
- Problem occurs on proxy path (`_run_agent_via_proxy`) — the Telegram Telegram channel.
- Local CLI (`hermes chat`) may not show this issue as clearly.

---

## Root Cause Analysis

### Two code paths in `gateway/run.py`

| Path | Function | Typing mechanism |
|---|---|---|
| Local CLI | `_run_agent()` (via `base.py`) | `_keep_typing` background loop in `base.py`, refreshes every 2s |
| Proxy (Telegram) | `_run_agent_via_proxy()` | `send_typing()` called **once** at line 12977 |

### Why this surfaced after the NKQ SSE streaming fix

Before the SSE streaming fix, requests via proxy were shorter and likely completed within 5 seconds, so the single `send_typing()` call was sufficient.

After the SSE streaming fix:
- SSE stream is real and responsive — requests can run much longer.
- Telegram typing indicator has a ~5s timeout (Telegram spec).
- After 5s, no refresh happens → typing disappears.
- The gap persists for the remainder of the long request.

### Code locations

**Proxy path (broken):** `gateway/run.py`, function `_run_agent_via_proxy`

```python
# Line ~12977 — ONE-TIME call, then done
await _adapter.send_typing(source.chat_id, metadata=_thread_metadata)
# ... SSE stream runs ...
# No more typing refresh → disappears after ~5s
```

**Local path (working):** `agent/platform_adapter/base.py`, `_run_agent()`

```python
# Line ~2794 — background loop
self._keep_typing(source.chat_id, _thread_metadata)
```

---

## Proposed Fix

Add a `_keep_typing` loop to `_run_agent_via_proxy()` in `gateway/run.py`, similar to how `base.py` does it.

### Where to patch

**File:** `/root/.hermes/hermes-agent/gateway/run.py`
**Function:** `_run_agent_via_proxy()`
**Target area:** Just before the SSE/stream handling block (~line 12970–13000)
**Cleanup:** Add to `finally` block (~line 13062)

### Fix snippet

Before the SSE stream handling (around line 12973):

```python
# Keep typing indicator alive during the SSE stream
_stop_typing = asyncio.Event()

async def _keep_typing_loop():
    while not _stop_typing.is_set():
        _typing_adapter = self.adapters.get(source.platform)
        if _typing_adapter:
            try:
                await _typing_adapter.send_typing(source.chat_id, metadata=_thread_metadata)
            except Exception:
                pass
        await asyncio.sleep(2.0)

_typing_task = asyncio.create_task(_keep_typing_loop())
```

In the `finally` block (around line 13062):

```python
finally:
    _stop_typing.set()
    if _typing_task:
        _typing_task.cancel()
        try:
            await _typing_task
        except asyncio.CancelledError:
            pass
    # ... existing cleanup ...
```

### Verification after fix

1. Send a message that triggers a long response (e.g., multi-step task).
2. Confirm typing indicator stays visible throughout the entire request.
3. Confirm typing stops after final message arrives.
4. Confirm no duplicate typing indicators appear.

---

## Why not fix in `base.py` or the adapter?

The typing loop in `base.py` is for local CLI and is already working correctly.
The proxy path bypasses `base.py`'s `_run_agent` and goes directly to `run.py`'s `_run_agent_via_proxy`.
The cleanest fix is in `run.py` since that's where the gap exists.

---

## Related files

- `/root/.hermes/hermes-agent/gateway/run.py` — where the fix goes
- `/root/.hermes/hermes-agent/agent/platform_adapter/base.py` — reference for working typing loop
- `/root/.hermes/workspaces/hermes-nkq-curlcffi-patch/` — NKQ SSE patch context
