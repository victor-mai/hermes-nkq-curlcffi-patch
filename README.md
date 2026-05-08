# Hermes NKQ curl_cffi Patch

Workspace này lưu lại bản vá Hermes để gọi NKQ Anthropic-compatible endpoint qua `curl_cffi` khi endpoint bị Cloudflare chặn TLS fingerprint của Python/httpx.

## Tên workspace

`/root/.hermes/workspaces/hermes-nkq-curlcffi-patch`

Lý do đặt tên:
- `hermes`: liên quan trực tiếp đến Hermes Agent.
- `nkq`: chỉ rõ endpoint/vendor đang cần workaround.
- `curlcffi`: chỉ rõ kỹ thuật dùng để vượt Cloudflare TLS fingerprint issue.
- `patch`: đây là bản vá cục bộ, cần re-apply sau khi Hermes update nếu upstream ghi đè file.

## Files trong folder này

- `anthropic_adapter.py`: bản clone hiện tại của file đã patch.
- `anthropic_adapter.patch`: git diff của thay đổi so với Hermes upstream checkout hiện tại, gồm curl_cffi + SSE streaming.
- `README.md`: tài liệu này.

## Vấn đề kỹ thuật

Từ cùng một VPS:

- `opencode run ...` gọi NKQ thành công.
- `hermes chat` trước khi patch bị lỗi HTTP 403 từ Cloudflare.

Kết luận: NKQ không block IP VPS. Vấn đề nằm ở tầng transport/TLS fingerprint của client Hermes Python.

Hermes Anthropic adapter mặc định dùng Anthropic SDK/httpx. Khi gọi `https://api.nkq.vn`, Cloudflare có thể nhận diện TLS ClientHello/fingerprint kiểu Python/httpx và trả 403/challenge.

Patch này thêm một client riêng dùng `curl_cffi` với:

```python
impersonate="safari17_0"
```

Điểm quan trọng: đây không chỉ là đổi HTTP header. `curl_cffi` thay đổi TLS ClientHello/fingerprint để giống Safari 17 hơn. Cloudflare nhìn thấy fingerprint giống browser hơn nên không chặn.

Bản mới cũng hỗ trợ **real Anthropic SSE streaming** cho NKQ:

- `messages.stream()` gửi `stream: true`.
- Parse `text/event-stream` từ `/v1/messages`.
- Yield lại event giống Anthropic SDK: `content_block_start`, `content_block_delta`, `message_delta`, `message_stop`.
- Reconstruct final message cho `get_final_message()`.
- Khi gặp `tool_use`, Hermes/Telegram có thể nhận event sớm để hiện progress thay vì chờ model generate xong toàn bộ.
- Nếu NKQ trả 2xx nhưng không phải SSE, fallback về blocking `create()` như bản cũ. Không fallback trên HTTP error như 429 để tránh double-spend request.

## File Hermes gốc cần patch

File runtime của Hermes:

```bash
/root/.hermes/hermes-agent/agent/anthropic_adapter.py
```

Sau khi Hermes update, file này có thể bị upstream ghi đè. Khi đó cần restore/re-apply patch từ workspace này.

## Dependency cần có

`curl_cffi` phải được cài trong Python environment mà Hermes đang chạy.

Hiện tại đã thấy package ở:

```bash
/root/.hermes/hermes-agent/venv/lib/python3.11/site-packages/curl_cffi/
```

Nếu sau update/venv recreate bị mất, cài lại bằng một trong các cách sau:

```bash
cd /root/.hermes/hermes-agent
source venv/bin/activate
uv pip install curl_cffi
```

Hoặc nếu không dùng `uv`:

```bash
cd /root/.hermes/hermes-agent
source venv/bin/activate
pip install curl_cffi
```

## Config liên quan

Config chính:

```bash
/root/.hermes/config.yaml
```

Các key quan trọng hiện tại:

```yaml
model:
  default: claude-opus-4-6
  provider: anthropic
  base_url: https://api.nkq.vn
```

Lưu ý:
- `provider: anthropic` để Hermes dùng Anthropic Messages adapter.
- `base_url: https://api.nkq.vn` là NKQ endpoint.
- Adapter tự append `/v1/messages` nếu base URL chưa có `/v1`.
- Không hardcode API key trong config nếu có thể tránh.

## Env liên quan

Env file chính:

```bash
/root/.hermes/.env
```

Key cần có cho Anthropic-compatible route hiện tại:

```bash
ANTHROPIC_API_KEY=...
```

Không commit/copy giá trị thật vào README hoặc patch.

Các key khác có thể tồn tại nhưng không liên quan trực tiếp đến NKQ Anthropic route:

```bash
GOOGLE_API_KEY=...
OPENROUTER_API_KEY=...
```

## Cách restore nhanh sau khi Hermes update

### Cách A — copy nguyên file đã clone

Dùng khi muốn khôi phục nhanh đúng bản đang chạy ổn:

```bash
cp /root/.hermes/workspaces/hermes-nkq-curlcffi-patch/anthropic_adapter.py    /root/.hermes/hermes-agent/agent/anthropic_adapter.py
```

Sau đó restart Hermes/gateway:

```bash
hermes gateway restart
```

Hoặc nếu đang dùng CLI thì thoát và mở lại `hermes chat`.

Nhược điểm: nếu upstream Hermes update nhiều trong `anthropic_adapter.py`, copy nguyên file có thể làm mất thay đổi mới của upstream.

### Cách B — apply patch bằng git

Dùng khi muốn giữ thay đổi upstream mới và chỉ apply phần NKQ workaround:

```bash
cd /root/.hermes/hermes-agent
git apply /root/.hermes/workspaces/hermes-nkq-curlcffi-patch/anthropic_adapter.patch
```

Nếu conflict/fail, mở diff và apply thủ công 3 phần chính:

1. Import:

```python
from types import SimpleNamespace
```

2. Thêm các class/function:

```python
_CurlCffiAnthropicStream
_CurlCffiAnthropicMessages
_CurlCffiAnthropicClient
_is_nkq_anthropic_endpoint
```

3. Trong `build_anthropic_client(...)`, sau `normalize_proxy_env_vars()`, thêm nhánh:

```python
if _is_nkq_anthropic_endpoint(base_url):
    return _CurlCffiAnthropicClient(api_key, base_url, timeout)
```

## Streaming behavior

Bản patch mới đã test được NKQ streaming thật:

```text
content-type: text/event-stream; charset=utf-8
event: content_block_start  # thinking/text/tool_use
event: content_block_delta  # thinking_delta/text_delta/input_json_delta/signature_delta
event: message_delta
event: message_stop
```

Tool-call streaming test đã thấy:

```text
event content_block_start block tool_use get_weather
event content_block_delta delta input_json_delta {"city": "Hanoi"}
final stop: tool_use
final block: tool_use get_weather {'city': 'Hanoi'}
```

Điều này giải quyết silent gap do bản cũ fake stream nhưng `__iter__` rỗng.

## Cách verify sau khi restore

Chạy smoke test từ Hermes checkout:

```bash
cd /root/.hermes/hermes-agent
set -a; [ -f /root/.hermes/.env ] && . /root/.hermes/.env; set +a
python - <<'PY'
import os
from agent.anthropic_adapter import build_anthropic_client

client = build_anthropic_client(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    base_url="https://api.nkq.vn",
    timeout=120,
)
resp = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=32,
    messages=[{"role": "user", "content": "Reply exactly: HERMES_NKQ_OK"}],
)
print(resp.content[0].text if hasattr(resp.content[0], 'text') else resp.content)
PY
```

Expected:

```text
HERMES_NKQ_OK
```

Nếu lỗi `ImportError: curl_cffi is required...` thì cài lại `curl_cffi` trong venv Hermes.

Nếu vẫn HTTP 403/Cloudflare thì kiểm tra:

- `model.base_url` có đúng `https://api.nkq.vn` không.
- Hermes đang chạy đúng checkout `/root/.hermes/hermes-agent` không.
- Gateway/CLI đã restart sau khi sửa code chưa.
- NKQ/Cloudflare có đổi rule mới không.

## Vì sao không dùng giống OpenCode?

OpenCode chạy trên Node.js và dùng stack HTTP/TLS khác. Từ cùng VPS, OpenCode đi được chứng minh IP không bị block. Nhưng Hermes là Python, mặc định dùng Anthropic SDK/httpx nên TLS fingerprint khác. `curl_cffi` là workaround phù hợp trong Python để impersonate browser TLS fingerprint.

## Rủi ro cần nhớ

- `curl_cffi` là package trong venv/site-packages: Hermes update thường không xóa trực tiếp, nhưng recreate venv thì có thể mất.
- `anthropic_adapter.py` và `gateway/run.py` là file trong Hermes source: Hermes update rất có thể ghi đè cả hai.
- Patch hiện tại chỉ activate khi `base_url` chứa `api.nkq.vn`; các endpoint Anthropic khác vẫn dùng client mặc định.
- Stream hiện tại là real SSE streaming qua `curl_cffi` nếu NKQ trả `text/event-stream`; có fallback blocking nếu endpoint trả 2xx non-SSE.

## Fix: Telegram Typing Indicator Disappears During Long Requests

**Ngày:** 2026-05-08
**Tình trạng:** ✅ Đã fix và verify — typing indicator giờ ổn định trong suốt request dài.

### Vấn đề

Trên proxy path (Telegram), `_run_agent_via_proxy()` trong `gateway/run.py` chỉ gửi `send_typing()` **1 lần duy nhất** → Telegram typing tự hết sau ~5s → indicator biến mất trong khi SSE stream còn đang chạy.

Local CLI path (`_run_agent` qua `base.py`) không gặp vấn đề này vì có `_keep_typing` loop refresh mỗi 2s.

### Root cause

Sau khi fix SSE streaming (NKQ patch), request dài hơn → Telegram typing hết timeout sau 5s → không có loop refresh → typing biến mất.

### Cách fix

Thêm `_keep_typing` loop vào `_run_agent_via_proxy()` trong `gateway/run.py`:

**Patch file:** `typing-fix.patch`

**Thay đổi 1 — thêm loop trước SSE stream (line ~12981):**

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

**Thay đổi 2 — cleanup trong `finally` block (line ~13075):**

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

### Checklist sau Hermes update

1. Kiểm tra file có còn patch không:

```bash
cd /root/.hermes/hermes-agent
grep -n "curl_cffi\|_CurlCffiAnthropic\|api.nkq.vn" agent/anthropic_adapter.py
grep -n "_keep_typing_loop\|_stop_typing" gateway/run.py
```

2. Nếu `anthropic_adapter.py` không có → restore bằng Cách A hoặc B.
3. Nếu `gateway/run.py` không có typing loop → apply `typing-fix.patch`:

```bash
cd /root/.hermes/hermes-agent
git apply /root/.hermes/workspaces/hermes-nkq-curlcffi-patch/typing-fix.patch
```

4. Kiểm tra `curl_cffi`:

```bash
cd /root/.hermes/hermes-agent
source venv/bin/activate
python -c "import curl_cffi; print(curl_cffi.__file__)"
```

5. Kiểm tra config:

```bash
hermes config | grep -A5 '^model:'
```

6. Chạy smoke test `HERMES_NKQ_OK`.
7. Restart gateway hoặc CLI.
