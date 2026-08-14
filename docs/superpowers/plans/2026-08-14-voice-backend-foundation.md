# Voice Backend Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make copied `speech-to-speech` snapshot run Vietnamese realtime voice sessions through server-side Google adapters and authenticated gateway.

**Architecture:** Keep upstream VAD, Smart Turn, OpenAI Realtime protocol, queues, and cancellation unchanged. Add isolated provider interfaces under `src/speech_to_speech/providers/google/`; application gateway owns anonymous session credentials, rate limits, health endpoint, and authenticated WebSocket proxy. Memory persistence and Android app are later plans.

**Tech Stack:** Python 3.10+, FastAPI, Uvicorn, Pydantic v2, SQLite standard library, Google Cloud Speech-to-Text V2, Google Gen AI SDK, pytest, pytest-asyncio, ruff, mypy.

## Global Constraints

- Copy snapshot source from `D:\PROJECT\Agentic-AI\voice-AI\Voice-chat\speech-to-speech-0.2.12`; do not include source `.git` metadata.
- Use `vi-VN` for STT and TTS.
- STT model: Google Cloud Speech-to-Text V2 `chirp_3`.
- LLM model: Gemini `gemini-2.5-flash`.
- Summary/fact model: Gemini `gemini-2.5-flash-lite`; not used by this plan.
- TTS model: Google Cloud Text-to-Speech `gemini-2.5-flash-tts`.
- Model IDs, endpoints, project, location, and voice remain server configuration, never Android constants.
- Retain upstream VAD, Smart Turn, pipeline queues, OpenAI Realtime events, cancellation, and stale-output filtering.
- Do not use Parakeet TDT or Qwen3-TTS on Vietnamese production path.
- Google credentials remain server-side. Never log credentials, raw audio, or full prompts.
- Gateway validates installation ID, rate-limits by installation ID and IP, and issues short-lived credentials.
- TLS terminates before production gateway. Never expose upstream realtime server publicly.
- Parameterize all SQLite queries. Raw audio persistence remains disabled.
- One backend instance, one to two concurrent sessions.
- No LiveKit, Node.js, Deepgram, ElevenLabs, vector DB, embeddings, payments, analytics, or custom media transport.

---

## File Structure

- `src/speech_to_speech/providers/google/config.py`: validated server-only Google configuration.
- `src/speech_to_speech/providers/google/stt.py`: Google STT stream mapped to upstream `PartialTranscription` and `Transcription`.
- `src/speech_to_speech/providers/google/llm.py`: Gemini stream mapped to provider-neutral text chunks.
- `src/speech_to_speech/providers/google/tts.py`: Gemini TTS stream mapped to PCM `numpy.ndarray` chunks.
- `src/speech_to_speech/gateway/app.py`: FastAPI factory, health, credential endpoint, authenticated proxy.
- `src/speech_to_speech/gateway/auth.py`: installation-ID parsing and signed short-lived credential validation.
- `src/speech_to_speech/gateway/rate_limit.py`: in-memory per-IP and per-installation fixed-window limiter.
- `src/speech_to_speech/gateway/settings.py`: gateway environment configuration and startup validation.
- `config/voice-realtime.example.json`: local production-shaped pipeline configuration without secrets.
- `tests/providers/google/`: fake-client provider adapter contract tests.
- `tests/gateway/`: credential, validation, rate-limit, health, and WebSocket proxy tests.

### Task 1: Make copied snapshot baseline reproducible

**Files:**
- Modify: `.gitignore`
- Modify: `pyproject.toml`
- Create: `config/voice-realtime.example.json`
- Test: `tests/test_voice_realtime_config.py`

**Interfaces:**
- Produces: JSON configuration consumed by `speech-to-speech config/voice-realtime.example.json`.
- Produces: optional dependency group `google`.

- [ ] **Step 1: Write failing config test**

```python
import json
from pathlib import Path


def test_voice_realtime_example_has_vietnamese_google_defaults():
    config = json.loads(Path("config/voice-realtime.example.json").read_text(encoding="utf-8"))
    assert config["stt"] == "google"
    assert config["tts"] == "google"
    assert config["llm_backend"] == "google-gemini"
    assert config["google_stt_language_code"] == "vi-VN"
    assert config["google_tts_language_code"] == "vi-VN"
    assert config["google_stt_model"] == "chirp_3"
    assert config["google_llm_model"] == "gemini-2.5-flash"
    assert config["google_tts_model"] == "gemini-2.5-flash-tts"
```

- [ ] **Step 2: Run failing test**

Run: `uv run pytest tests/test_voice_realtime_config.py -v`

Expected: FAIL because config file does not exist.

- [ ] **Step 3: Add Google extras and safe config**

Add this dependency group to `pyproject.toml`:

```toml
[project.optional-dependencies]
google = [
    "google-cloud-speech>=2.31.0",
    "google-genai>=1.0.0",
]
```

Create `config/voice-realtime.example.json` with non-secret values:

```json
{
  "mode": "realtime",
  "num_pipelines": 2,
  "stt": "google",
  "tts": "google",
  "llm_backend": "google-gemini",
  "google_project": "set-GOOGLE_CLOUD_PROJECT",
  "google_location": "global",
  "google_stt_model": "chirp_3",
  "google_stt_language_code": "vi-VN",
  "google_llm_model": "gemini-2.5-flash",
  "google_tts_model": "gemini-2.5-flash-tts",
  "google_tts_language_code": "vi-VN",
  "google_tts_voice": "set-after-vietnamese-voice-test"
}
```

Add `.env`, `*.db`, `*.sqlite3`, `.pytest_cache/`, `__pycache__/`, `.mypy_cache/`, and `src/speech_to_speech/tmp/` to `.gitignore` if absent.

- [ ] **Step 4: Run config test**

Run: `uv run pytest tests/test_voice_realtime_config.py -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add .gitignore pyproject.toml config/voice-realtime.example.json tests/test_voice_realtime_config.py
git commit -m "chore: add Vietnamese Google voice config"
```

### Task 2: Add provider configuration and Google adapter contracts

**Files:**
- Create: `src/speech_to_speech/providers/__init__.py`
- Create: `src/speech_to_speech/providers/google/__init__.py`
- Create: `src/speech_to_speech/providers/google/config.py`
- Create: `src/speech_to_speech/providers/google/stt.py`
- Create: `src/speech_to_speech/providers/google/llm.py`
- Create: `src/speech_to_speech/providers/google/tts.py`
- Test: `tests/providers/google/test_config.py`
- Test: `tests/providers/google/test_stt.py`
- Test: `tests/providers/google/test_llm.py`
- Test: `tests/providers/google/test_tts.py`

**Interfaces:**
- Consumes: `numpy.ndarray` PCM16 mono input at 16 kHz.
- Produces: `PartialTranscription`, `Transcription`, streamed text strings, and PCM16 bytes.
- Produces: `GoogleVoiceConfig.from_env(env: Mapping[str, str]) -> GoogleVoiceConfig`.

- [ ] **Step 1: Write failing configuration tests**

```python
import pytest
from speech_to_speech.providers.google.config import GoogleVoiceConfig


def test_config_reads_vietnamese_production_defaults():
    config = GoogleVoiceConfig.from_env({"GOOGLE_CLOUD_PROJECT": "voice-project"})
    assert config.project == "voice-project"
    assert config.stt_language_code == "vi-VN"
    assert config.stt_model == "chirp_3"
    assert config.llm_model == "gemini-2.5-flash"
    assert config.tts_model == "gemini-2.5-flash-tts"


def test_config_rejects_missing_project():
    with pytest.raises(ValueError, match="GOOGLE_CLOUD_PROJECT"):
        GoogleVoiceConfig.from_env({})
```

- [ ] **Step 2: Implement minimal immutable config**

```python
from dataclasses import dataclass
from typing import Mapping


@dataclass(frozen=True)
class GoogleVoiceConfig:
    project: str
    location: str = "global"
    stt_model: str = "chirp_3"
    stt_language_code: str = "vi-VN"
    llm_model: str = "gemini-2.5-flash"
    tts_model: str = "gemini-2.5-flash-tts"
    tts_language_code: str = "vi-VN"
    tts_voice: str | None = None

    @classmethod
    def from_env(cls, env: Mapping[str, str]) -> "GoogleVoiceConfig":
        project = env.get("GOOGLE_CLOUD_PROJECT", "").strip()
        if not project:
            raise ValueError("Missing required configuration: GOOGLE_CLOUD_PROJECT")
        return cls(project=project, location=env.get("GOOGLE_CLOUD_LOCATION", "global"))
```

- [ ] **Step 3: Write adapter contract tests using fakes**

Cover these exact cases:

```python
# STT: partial emitted before final; final has language_code == "vi-VN".
# LLM: fake stream ["Xin ", "chào"] yields ["Xin ", "chào"].
# TTS: fake PCM chunks [b"\x00\x00", b"\x01\x00"] preserve byte order.
# Adapter construction receives no credential argument and does not log config.
```

Use injected protocol clients. Tests must not import Google SDKs or call network.

- [ ] **Step 4: Implement thin injected adapters**

Define protocols and adapters with these callable contracts:

```python
class GoogleSttClient(Protocol):
    def stream_pcm16(self, audio: bytes, sample_rate_hz: int, language_code: str, model: str) -> Iterable[tuple[str, bool]]: ...

class GoogleGeminiClient(Protocol):
    def stream_text(self, model: str, prompt: str) -> Iterable[str]: ...

class GoogleTtsClient(Protocol):
    def stream_pcm16(self, model: str, language_code: str, voice: str | None, text: str) -> Iterable[bytes]: ...
```

`GoogleSttAdapter.transcribe(audio: np.ndarray) -> Iterator[PartialTranscription | Transcription]` converts PCM16 once and maps `is_final=False` to partial and `is_final=True` to final. `GoogleGeminiAdapter.stream(prompt: str) -> Iterator[str]` drops empty chunks. `GoogleTtsAdapter.synthesize(text: str) -> Iterator[np.ndarray]` converts PCM16 little-endian bytes to `np.int16` arrays. Keep SDK construction in factory functions invoked only at server startup.

- [ ] **Step 5: Run provider contract tests**

Run: `uv run pytest tests/providers/google -v`

Expected: PASS without Google credentials or network access.

- [ ] **Step 6: Commit**

```bash
git add src/speech_to_speech/providers tests/providers/google
git commit -m "feat: add Google voice provider adapters"
```

### Task 3: Wire Google adapters into upstream pipeline selection

**Files:**
- Modify: `src/speech_to_speech/arguments_classes/module_arguments.py`
- Create: `src/speech_to_speech/arguments_classes/google_voice_arguments.py`
- Modify: `src/speech_to_speech/s2s_pipeline.py`
- Test: `tests/test_google_voice_pipeline_selection.py`

**Interfaces:**
- Consumes: `--stt google --llm_backend google-gemini --tts google` or equivalent JSON fields.
- Produces: upstream handler chain using Google adapters and existing `CancelScope`.

- [ ] **Step 1: Write failing parser/selection tests**

```python
def test_google_voice_config_selects_google_handlers(monkeypatch):
    monkeypatch.setattr("sys.argv", ["speech-to-speech", "config/voice-realtime.example.json"])
    args = parse_arguments()
    assert args.module_kwargs.stt == "google"
    assert args.module_kwargs.llm_backend == "google-gemini"
    assert args.module_kwargs.tts == "google"
```

Add a second test asserting invalid Google configuration raises `ValueError` before handler threads start.

- [ ] **Step 2: Extend argument literals and dataclass**

Add `"google"` STT/TTS choices and `"google-gemini"` LLM choice. Create `GoogleVoiceHandlerArguments` with fields matching `GoogleVoiceConfig`, and defaults copied from Task 2.

- [ ] **Step 3: Add three pipeline handler wrappers**

Wrap Task 2 adapters in upstream handler interfaces: `BaseSTTHandler` for STT; `BaseHandler[LLMIn, LLMOut]` for LLM; `BaseHandler[TTSIn, TTSOut]` for TTS. Each wrapper must check `cancel_scope` before producing each output and tag TTS output with incoming `cancel_generation`.

- [ ] **Step 4: Register handlers in pipeline builder**

Update `ParsedArguments`, `parse_arguments`, `prepare_all_args`, and `_build_pipeline_handlers` to construct Google handlers only for selected Google modes. Pass a shared `GoogleVoiceConfig`; do not introduce a second queue system or protocol.

- [ ] **Step 5: Run focused and upstream cancellation tests**

Run: `uv run pytest tests/test_google_voice_pipeline_selection.py tests/openai_realtime/test_realtime_service.py -v`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/speech_to_speech/arguments_classes src/speech_to_speech/s2s_pipeline.py tests/test_google_voice_pipeline_selection.py
git commit -m "feat: wire Google adapters into realtime pipeline"
```

### Task 4: Add anonymous credential and rate-limit gateway

**Files:**
- Create: `src/speech_to_speech/gateway/__init__.py`
- Create: `src/speech_to_speech/gateway/settings.py`
- Create: `src/speech_to_speech/gateway/auth.py`
- Create: `src/speech_to_speech/gateway/rate_limit.py`
- Create: `src/speech_to_speech/gateway/app.py`
- Test: `tests/gateway/test_auth.py`
- Test: `tests/gateway/test_rate_limit.py`
- Test: `tests/gateway/test_app.py`

**Interfaces:**
- `POST /v1/sessions` accepts `{"installation_id": "<UUID>"}`.
- Returns `{"credential": "<signed-token>", "expires_at": "<UTC ISO-8601>"}`.
- `GET /healthz` returns `{"status": "ok"}`.
- `WS /v1/realtime` requires `Authorization: Bearer <credential>` and proxies only valid clients to private upstream URL.

- [ ] **Step 1: Write failing auth tests**

```python
from uuid import uuid4
from speech_to_speech.gateway.auth import SessionCredentialSigner


def test_signer_round_trips_valid_installation_id():
    signer = SessionCredentialSigner(secret=b"x" * 32, ttl_seconds=300)
    token = signer.issue(str(uuid4()))
    assert signer.verify(token).installation_id


def test_signer_rejects_tampered_token():
    signer = SessionCredentialSigner(secret=b"x" * 32, ttl_seconds=300)
    token = signer.issue(str(uuid4())) + "x"
    assert signer.verify(token) is None
```

Add tests that reject non-UUID IDs, blank IDs, expired credentials, and credentials signed with another secret.

- [ ] **Step 2: Implement dependency-free signed credentials**

Use `hmac`, `hashlib`, `base64`, `json`, `time`, and `uuid` only. Token payload contains `installation_id` and integer `exp`; sign URL-safe base64 payload using SHA-256 HMAC. `verify()` must compare signatures with `hmac.compare_digest`, validate UUID, and reject expired payloads.

- [ ] **Step 3: Write failing rate-limit tests**

```python
def test_limiter_blocks_third_request_in_window():
    limiter = FixedWindowRateLimiter(limit=2, window_seconds=60, clock=lambda: 100.0)
    assert limiter.allow("installation:abc")
    assert limiter.allow("installation:abc")
    assert not limiter.allow("installation:abc")
```

Add independent-key and window-expiry tests.

- [ ] **Step 4: Implement bounded in-memory fixed-window limiter**

Expose `FixedWindowRateLimiter.allow(key: str) -> bool`. Keep only current-window keys and prune expired windows each call. Use `threading.Lock`; no external cache.

- [ ] **Step 5: Write failing FastAPI endpoint tests**

Use `fastapi.testclient.TestClient`. Cover successful session issue, malformed payload `422`, bad installation `400`, installation cap `429`, IP cap `429`, `healthz`, missing WebSocket authorization close code `1008`, and invalid credential close code `1008`.

- [ ] **Step 6: Implement settings and app factory**

`GatewaySettings.from_env()` requires `GATEWAY_SESSION_SECRET` with at least 32 UTF-8 bytes and `REALTIME_UPSTREAM_WS_URL` using `ws://` or `wss://`; reject missing values at startup. `create_app(settings, websocket_connector)` adds no credential/raw-audio/full-prompt logging. Proxy forwards text/binary frames both ways only after credential verification. Connector injection enables in-memory fake WebSocket tests.

- [ ] **Step 7: Run gateway tests**

Run: `uv run pytest tests/gateway -v`

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add src/speech_to_speech/gateway tests/gateway
git commit -m "feat: add anonymous realtime session gateway"
```

### Task 5: Add executable startup, observability boundaries, and regression suite

**Files:**
- Create: `src/speech_to_speech/gateway/__main__.py`
- Create: `src/speech_to_speech/observability.py`
- Modify: `README.md`
- Test: `tests/gateway/test_settings.py`
- Test: `tests/test_observability.py`

**Interfaces:**
- `python -m speech_to_speech.gateway` starts gateway only after complete settings validation.
- `VoiceMetrics.record(event: str, **fields: float | int | str)` emits allowlisted structured fields.

- [ ] **Step 1: Write failing observability test**

```python
from speech_to_speech.observability import VoiceMetrics


def test_metrics_redacts_sensitive_fields(caplog):
    VoiceMetrics().record("request_complete", model="gemini-2.5-flash", input_tokens=12, prompt="secret", audio=b"pcm")
    assert "gemini-2.5-flash" in caplog.text
    assert "secret" not in caplog.text
    assert "pcm" not in caplog.text
```

- [ ] **Step 2: Implement allowlisted structured metrics**

Allow only `model`, `input_tokens`, `output_tokens`, `estimated_cost`, `latency_ms`, `tts_first_audio_ms`, `stt_final_ms`, `cancellation_ms`, `generation_id`, and `retrieval_id`. Drop unknown fields. Do not add analytics transport.

- [ ] **Step 3: Add gateway module entrypoint**

```python
import uvicorn
from speech_to_speech.gateway.app import create_app
from speech_to_speech.gateway.settings import GatewaySettings

settings = GatewaySettings.from_env()
uvicorn.run(create_app(settings), host=settings.host, port=settings.port)
```

Keep startup validation before Uvicorn begins accepting connections.

- [ ] **Step 4: Document local setup without secrets**

Add exact commands:

```bash
uv sync --group dev --extra google
copy config\voice-realtime.example.json config\voice-realtime.local.json
set GOOGLE_CLOUD_PROJECT=your-project
set GATEWAY_SESSION_SECRET=replace-with-32-or-more-secret-bytes
set REALTIME_UPSTREAM_WS_URL=ws://127.0.0.1:8765/v1/realtime
python -m speech_to_speech.gateway
```

Document never committing `config/voice-realtime.local.json`, service-account files, or `.env`.

- [ ] **Step 5: Run quality gates**

Run: `uv run ruff check src tests && uv run mypy src && uv run pytest -q`

Expected: all commands exit `0`.

- [ ] **Step 6: Commit**

```bash
git add README.md src/speech_to_speech/gateway/__main__.py src/speech_to_speech/observability.py tests/gateway/test_settings.py tests/test_observability.py
git commit -m "feat: add secure voice gateway startup"
```

## Manual Acceptance

1. Configure Google credentials only in server environment.
2. Start private upstream realtime server with `config/voice-realtime.local.json`.
3. Start gateway with private upstream URL.
4. Call `POST /v1/sessions` with generated UUID and connect using returned credential.
5. Send 16 kHz mono PCM16 frames and verify partial/final Vietnamese transcript.
6. Interrupt playback; verify existing upstream cancellation and stale-output filtering stop old output.
7. Confirm logs contain only allowlisted metrics.
8. Verify `git status --ignored` shows local config and credentials ignored.

## Plan Self-Review

- Spec coverage for snapshot use, Google models, Vietnamese locale, OpenAI-compatible upstream protocol, gateway authentication/rate limits, credentials, cancellation preservation, observability, and provider/gateway tests: Tasks 1–5.
- Deliberately deferred: SQLite memory/context/budget/retention and Android client. These require stable gateway and provider contracts from this plan.
- No placeholders, undefined plan APIs, raw-audio persistence, client credentials, custom queues, or new transport.
