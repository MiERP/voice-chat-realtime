# Voice Chat Realtime MVP Design

Date: 2026-08-13
Status: Approved design

## Goal

Build an Android native voice companion with natural, interruptible Vietnamese conversation. MVP supports anonymous users, one to two concurrent users, Hugging Face `speech-to-speech` as voice backend, Google Cloud AI APIs, and SQLite memory keyed by anonymous device ID.

## Scope

Included:

- Android native client.
- Hugging Face `speech-to-speech` Python backend.
- Realtime audio over an OpenAI Realtime-compatible WebSocket protocol.
- Google Cloud STT, LLM, and TTS adapters.
- Vietnamese speech support.
- Barge-in and stale-output cancellation.
- Anonymous device identity.
- In-session memory, long-term summaries, SQLite FTS5 retrieval, and token budgets.
- Tests for cancellation, persistence, context composition, and reconnect behavior.

Excluded:

- Account registration and login.
- Raw audio persistence by default.
- LiveKit, Node.js backend, Deepgram, ElevenLabs, and custom media transport.
- Vector database, embeddings, payments, analytics, multi-agent workflows, and custom model hosting.

## Reference repositories

Use these repositories as patterns, not blind copies:

- `D:\PROJECT\Agentic-AI\voice-AI\Voice-chat\speech-to-speech-0.2.12`
  - Hugging Face `speech-to-speech` source snapshot.
  - Stage-separated VAD, STT, LLM, and TTS pipeline.
  - OpenAI Realtime-compatible WebSocket and WebRTC event protocol.
  - Generation-based cancellation, stale-output filtering, Smart Turn, and bounded history.
- `https://github.com/huggingface/speech-to-speech`
  - Upstream implementation and current protocol behavior.
- `https://huggingface.co/spaces/pollen-robotics/reachy_mini_conversation_app`
  - Realtime client pattern, PCM audio flow, server VAD, `interrupt_response`, output queue clearing, and conversation states.
- `D:\PROJECT\Agentic-AI\voice-AI\Voice-chat\agent-starter-node-main`
  - Reference only for voice interaction behavior and tests. Do not copy LiveKit or Node runtime.
- `D:\PROJECT\Agentic-AI\voice-AI\Voice-chat\agent-starter-react-main`
  - Reference only for client state handling. Do not copy React or token route.

Do not reimplement VAD, streaming audio queues, turn revisions, or OpenAI Realtime events already supplied by `speech-to-speech`. Add only Google adapters, authenticated session gateway, Android client, and durable memory.

## Architecture

```text
Android native
  │ PCM audio over authenticated WebSocket
  ▼
Application gateway
  │ validates anonymous device session and rate limits
  ▼
Hugging Face speech-to-speech server
  ├─ VAD and Smart Turn
  ├─ Google Cloud Speech-to-Text adapter
  ├─ Google Gemini LLM adapter
  ├─ Google Gemini Text-to-Speech adapter
  ├─ response cancellation and stale-output filtering
  └─ conversation service
       └─ SQLite
```

Android speaks the OpenAI Realtime-compatible event protocol to the server. The gateway creates short-lived session authorization, retains Google credentials server-side, and prevents direct public exposure of the raw `speech-to-speech` server. The Android app never contains Google service-account credentials or permanent server tokens.

## Model configuration

Initial production configuration:

| Stage | Model | Responsibility |
| --- | --- | --- |
| STT | Google Cloud Speech-to-Text V2 `chirp_3` | Streaming Vietnamese transcript and turn text. |
| LLM | Gemini `gemini-2.5-flash` | Fast companion replies, streamed into TTS. |
| Summary and fact extraction | Gemini `gemini-2.5-flash-lite` | Background memory updates with small output limits. |
| TTS | Google Cloud Text-to-Speech `gemini-2.5-flash-tts` | Streaming Vietnamese PCM audio. |

Use `vi-VN` for STT and TTS. Test several Gemini TTS voices with Vietnamese speakers before choosing one default voice. API model names and endpoints remain configuration, not Android constants.

Local backend defaults in upstream `speech-to-speech` do not meet Vietnamese MVP requirements: Parakeet TDT does not list Vietnamese support, and Qwen3-TTS does not list Vietnamese support. Do not use either on the Vietnamese production path.

## Backend modules

Keep application additions small and isolated:

- `gateway/`: anonymous session authorization, validation, rate limit, health endpoint, and WebSocket proxy.
- `speech-to-speech configuration`: selected Google adapters, model IDs, locale, and endpoint settings.
- `memory/`: context composer, SQLite repositories, transcript cleanup, summary scheduling, and FTS5 retrieval.
- `observability/`: structured latency, token, cost, and cancellation events.
- `android/`: native client, microphone capture, audio playback, realtime protocol client, and UI state.

The speech server owns active voice pipeline state. The memory service builds LLM context only after a final user transcript. Provider SDK calls stay inside provider adapter implementations.

## Realtime protocol and audio

Use upstream OpenAI Realtime-compatible core events:

- Android sends `session.update`, `input_audio_buffer.append`, `response.create`, and `response.cancel`.
- Server emits speech-start/stop, transcript delta/final events, response text/audio deltas, tool events when enabled, and response completion.
- Android sends 16 kHz mono PCM signed 16-bit microphone frames unless server configuration requires another format.
- Server returns streamed PCM audio. Android plays it with `AudioTrack` and stops playback immediately when response cancellation begins.

The Android client tracks `listening`, `thinking`, `speaking`, `reconnecting`, and `error` states. It displays final and partial transcripts but persists only final transcript text.

## Conversation and memory

Anonymous identity:

- Android creates a random `installationId` at first launch.
- Store it in Android Keystore-backed local storage.
- Gateway validates length and format, rate-limits by installation and IP, and issues a short-lived session credential.
- `installationId` is pseudonymous and is not authentication.

Session memory:

- Keep active voice turns in current server session.
- Bound conversation history by token budget and recent-turn count.
- Prevent duplicate turn persistence during reconnect.

Long-term memory:

- Store summaries and validated profile facts by `installationId`.
- Generate summary at session end or after configured turn threshold.
- Summary or fact failure never delays voice reply.
- Do not store raw audio by default.

SQLite core tables:

- `devices`: installation ID, created time, last seen time, retention metadata.
- `conversations`: session ID, installation ID, start/end time, status.
- `turns`: session ID, sequence, speaker, raw text, cleaned text, timestamps, generation ID, final state.
- `session_summaries`: session ID, text, covered sequence, token estimate, created time.
- `memories`: installation ID, key, value, type, confidence, source turn ID, updated time.
- FTS5 virtual tables over summary and memory text.

Use SQLite WAL mode, parameterized queries, migrations, and bounded retention. SQLite supports one backend instance. Move to shared persistence only after real multi-instance deployment.

## Token-efficient context

Do not replay full history. Compose final user turn context as:

```text
stable companion policy + profile facts + active summary + recent clean turns + FTS5 retrieval + current transcript
```

Initial input budget is 8,000 tokens with 1,000 tokens reserved for reply:

- Stable policy and safety rules: maximum 1,500 tokens.
- Profile facts: maximum 800 tokens.
- Active session summary: maximum 1,500 tokens.
- Recent clean turns: latest 6 turns, maximum 3,000 tokens.
- SQLite FTS5 retrieval: top 3 items, maximum 1,000 tokens.
- Current transcript and request overhead: remaining budget.

If input exceeds budget, remove retrieval first, then reduce recent turns from 6 to 4, then reduce summary from 1,500 to 800 tokens. Never remove consent, safety, identity, or explicit user preferences.

Transcript handling:

- Persist final STT transcript only.
- Normalize whitespace, duplicate fillers, and punctuation deterministically.
- Preserve names, numbers, negation, questions, and emotional words.
- Merge adjacent same-speaker utterances separated by less than two seconds.
- Discard empty turns. Mark low-confidence content unclear rather than inventing correction.
- Do not call an LLM to clean transcript in voice-response hot path.

Incremental memory:

- Summarize turns after `coveredSequence` every 8 final turns or when recent-turn context exceeds 3,000 tokens.
- Advance `coveredSequence` atomically only after summary write succeeds.
- Extract candidate profile facts asynchronously using `gemini-2.5-flash-lite`, strict structured output, source turn ID, type, confidence, and updated time.
- Validate fact schema before upsert.
- Retrieve relevant summary and memory with FTS5 from current cleaned transcript.
- Add embeddings only after measured recall failures justify them.

Cost control:

- Keep static companion policy and schemas stable and versioned.
- Log model, tokens, estimated cost, prompt-section sizes, request latency, TTS first-audio timing, STT final timing, cancellation time, and retrieval IDs.
- Apply per-installation daily cost cap.
- Before cap is reached, reduce context using budget policy. After cap is reached, return a clear retry-later message and do not call paid models.
- Use `gemini-2.5-flash-lite` only for background summary and fact jobs.

## Interruption and latency

Every assistant response has a monotonically increasing `generationId`.

When user speech begins during assistant playback:

1. Mark active generation cancelled.
2. Send or apply `response.cancel`.
3. Stop Android `AudioTrack` playback and clear queued audio.
4. Drop late audio and text chunks from stale generation.
5. Continue with final transcript for new turn.

Use server VAD with interruption enabled. Keep Smart Turn and turn revision behavior from `speech-to-speech`. Do not add speculative generation changes before baseline tests.

Latency targets:

- First assistant audio at or below 1.5 seconds after user turn completes.
- User speech start to playback stop below 250 ms.
- Reconnect restores session without duplicate persisted turns.

Measure microphone final frame, VAD end, STT partial/final, LLM first token, TTS request, TTS first audio, Android playback start, and cancellation stop time.

## Android client

Use Kotlin and Android native audio APIs:

- Request microphone permission.
- Create and persist anonymous installation ID.
- Request short-lived session credential from gateway.
- Capture 16 kHz mono PCM with `AudioRecord`.
- Send realtime protocol events over secure WebSocket.
- Play PCM with `AudioTrack`.
- Show listening, thinking, speaking, reconnecting, and error state.
- Show transcript, mute, and end-session controls.
- Reconnect with bounded retry and visible error state.
- Never embed Google credentials or permanent server secrets.

MVP stays one conversation screen. No account, avatar, camera, screen sharing, or payment UI.

## Security and privacy

- Put Google credentials only in server secret storage.
- Gateway validates all session inputs and applies per-installation and per-IP rate limits.
- Require TLS for production WebSocket traffic.
- Do not expose upstream `speech-to-speech` server directly to the public internet.
- Do not log credentials, raw audio, or full prompts.
- Minimize transcript logs and support message/summary retention settings.
- Use parameterized SQLite queries.
- Treat installation ID as pseudonymous data.

## Failure handling

- Google STT timeout: cancel active response, keep session alive, and allow another turn.
- Gemini LLM failure: cancel TTS, report short voice-safe error, and keep session alive.
- Gemini TTS failure: show assistant text and allow next turn.
- SQLite summary failure: log structured error and retry later without delaying voice reply.
- WebSocket disconnect: reconnect with backoff and deduplicate turn writes by session ID, sequence, and generation ID.
- Invalid configuration: fail startup and name missing configuration.

## Verification

Backend tests must cover:

- Google provider adapter contracts with fake streams.
- Partial and final transcript behavior.
- Generation cancellation and stale chunk filtering.
- Barge-in stopping queued audio.
- SQLite migrations and repository operations.
- Token budget degradation order.
- Incremental summary `coveredSequence` behavior.
- Summary failure isolation.
- Anonymous session validation, rate limits, and credential scope.
- Reconnect without duplicate turns.

Android tests must cover:

- Installation ID creation and persistence.
- Permission and session credential failures.
- Realtime state transitions.
- PCM queue clear on barge-in.
- Reconnect behavior.

Manual acceptance test:

1. Start app with new installation ID.
2. Grant microphone permission.
3. Speak Vietnamese for several turns.
4. Interrupt assistant while it speaks.
5. Confirm old audio stops and a new response starts.
6. Restart gateway and backend.
7. Reconnect with same installation ID.
8. Confirm summary loads without raw-audio persistence.
9. Verify Google credentials are absent from APK and client logs.
10. Verify first assistant audio and interruption latency meet targets in 30 warm runs.

## Success criteria

- Anonymous Android user has working Vietnamese voice conversation.
- Speech server uses Google Cloud `chirp_3`, Gemini `gemini-2.5-flash`, and `gemini-2.5-flash-tts` through server-side configuration.
- First assistant audio meets 1.5-second target in local MVP measurements.
- Barge-in does not leak stale assistant audio.
- SQLite preserves history and long-term summary through ordinary backend restart.
- Context composer remains within configured budget.
- Secrets remain server-side.

## Explicit simplifications

- One backend instance and SQLite first.
- Hugging Face `speech-to-speech` owns realtime voice pipeline and protocol.
- No local Vietnamese STT/TTS in MVP.
- No vector retrieval, embeddings, account system, or provider failover until measurements justify them.
