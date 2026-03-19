# GPT-Line Realtime Bridge Service Repository Specification
**Service owner:** Realtime voice / AI media developer  
**Primary runtime:** Node.js 22 + TypeScript  
**Primary role:** Bridge live caller audio from the telephony system to OpenAI Realtime and return assistant audio back to the caller while emitting timing and lifecycle events to the Core API.

---

## 1. Mission

Build the production Realtime Bridge for GPT-Line. This repository must fully implement the live audio/session layer so a developer can work from this document alone.

The finished service must:

- accept live call start requests from the Telephony service
- create one OpenAI Realtime session per active call
- bridge bidirectional audio between Asterisk and OpenAI
- enforce the exact Hebrew assistant behavior described below
- support interruption / barge-in cleanly
- stop speaking immediately when Telephony ends the session
- track the absolute time cutoff received from Core preflight data
- emit all required lifecycle events and timing events back to the Core API
- expose health endpoints and session state
- never own billing truth
- never store payment information
- avoid storing transcripts or audio by default

This service is the only service allowed to communicate directly with OpenAI Realtime for GPT-Line voice calls.

---

## 2. Hard decisions already locked

Do not change these decisions.

- Runtime: **Node.js 22**
- Language: **TypeScript**
- HTTP framework: **Fastify**
- Realtime provider: **OpenAI Realtime**
- Session mode: **server-side realtime session controlled by this service**
- One active GPT session per `call_session_id`
- Primary account identifier passed through the system: `phone_e164`
- Primary live-session identifier: `call_session_id`
- Balance/cutoff truth comes from the Core API, not from this service
- This service must never speak directly to callers about balance in its own words
- No long-term transcript storage by default
- No raw payment data ever

---

## 3. Canonical shared enums used by this service

### 3.1 Call ended reason enum

Allowed values:

- `star_exit`
- `caller_hangup`
- `time_expired`
- `system_error`
- `backend_revoke`
- `openai_error`
- `bridge_error`
- `telephony_disconnect`

### 3.2 Bridge timing event enum

Allowed timing events emitted to Core:

- `bridge-warning-due`
- `bridge-cutoff-due`

---

## 4. External systems this repo must integrate with

This repo must integrate with:

1. **Telephony Service**
2. **Core API Service**
3. **OpenAI Realtime**

The exact contracts are specified here.

---

## 5. Overall role in the product

Caller flow summary:

1. The caller presses `1` on the phone menu
2. Telephony asks Core whether the call is allowed
3. If allowed, Telephony calls this service to start a bridge
4. This service creates the OpenAI Realtime session and begins streaming audio
5. During the conversation, this service:
   - forwards caller audio to OpenAI
   - forwards assistant audio back to the caller
   - handles interruptions
   - tracks warning and timeout thresholds
6. If the caller presses `*`, Telephony calls this service to stop
7. If time reaches the warning threshold, this service emits `bridge-warning-due` to Core
8. If time reaches zero, this service emits `bridge-cutoff-due` to Core and ends the AI side
9. Telephony later learns about the resulting command from Core and handles caller-facing prompts and menu return

---

## 6. Required repository output

The repository must include:

- application source code
- API server
- realtime session manager
- audio/media bridge implementation
- OpenAI adapter
- session timer subsystem
- event client for Core
- unit tests
- integration tests
- mocked OpenAI tests
- Dockerfile
- Docker Compose for local development
- `.env.example`
- README with exact run instructions
- operational runbook
- structured logging
- graceful shutdown logic

Do not leave core behavior as stubs.

---

## 7. Repository architecture

### 7.1 Required modules

```text
/
  README.md
  docs/
    architecture.md
    deployment.md
    operations.md
  src/
    server.ts
    config/
    routes/
    bridge/
    openai/
    audio/
    timers/
    events/
    logging/
    validation/
    types/
  test/
    unit/
    integration/
  Dockerfile
  docker-compose.yml
  .env.example
```

### 7.2 Internal design requirements

The service must include these internal responsibilities:

- `CallSessionRegistry`
- `OpenAISessionClient`
- `MediaBridge`
- `TimerSupervisor`
- `CoreApiEventClient`
- `BridgeController`
- `SessionCleanupManager`

---

## 8. Session identifiers and runtime fields

### 8.1 Primary identifiers

- `call_session_id`
- `phone_e164`

### 8.2 Required runtime session record

```ts
type ActiveBridgeSession = {
  callSessionId: string;
  phoneE164: string;
  absoluteCutoffEpochMs: number;
  warningAtSeconds: number;
  openaiSessionRef?: string;
  asteriskMediaIp: string;
  asteriskMediaPort: number;
  codec: 'pcmu';
  bridgeState: 'starting' | 'connected' | 'ending' | 'ended' | 'error';
  warningSent: boolean;
  connectedAt?: string;
  terminatedReason?: 'star_exit' | 'caller_hangup' | 'time_expired' | 'system_error' | 'backend_revoke' | 'openai_error' | 'bridge_error' | 'telephony_disconnect';
};
```

### 8.3 Persistence rule

This service may keep active state in memory and optionally mirror it in Redis, but it must not persist bridge state as business truth in PostgreSQL. Billing truth remains in Core.

---

## 9. HTTP API this service must expose

### 9.1 Start bridge

`POST /internal/bridge/start`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "absolute_cutoff_epoch_ms": 1773654468120,
  "warning_at_seconds": 60,
  "asterisk_media": {
    "ip": "10.0.2.11",
    "port": 40012,
    "codec": "pcmu"
  }
}
```

**Behavior**

1. Validate request
2. Reject if session already exists
3. Create an OpenAI Realtime session
4. Apply the required Hebrew system instructions
5. Create the media bridge to the supplied Asterisk media IP/port
6. Mark the session `connected`
7. Emit `bridge-connected` to Core
8. Return success

**Success**
```json
{
  "ok": true,
  "bridge_state": "connected",
  "openai_session_ref": "rt_sess_123456"
}
```

**Failure**
```json
{
  "ok": false,
  "error_code": "bridge_unavailable"
}
```

### 9.2 End bridge

`POST /internal/bridge/end`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "reason": "star_exit"
}
```

Allowed reasons are the canonical ended-reason enum.

**Behavior**

1. Look up the active session
2. Stop accepting caller audio
3. Cancel assistant audio immediately
4. Tear down media bridge
5. End the OpenAI session
6. Emit `bridge-ended` to Core
7. Release all resources

**Success**
```json
{
  "ok": true
}
```

### 9.3 Get session health

`GET /internal/bridge/health/:call_session_id`

**Response**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "bridge_state": "connected",
  "remaining_seconds_estimate": 54,
  "warning_sent": true
}
```

### 9.4 Generic health endpoint

`GET /health`

**Response**
```json
{
  "ok": true,
  "service": "gpt-line-realtime-bridge"
}
```

---

## 10. Core API event contracts consumed by this design

This service must send the following events to Core.

### 10.1 Bridge connected

`POST /internal/events/bridge-connected`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "connected_at": "2026-03-16T09:42:13.010Z"
}
```

### 10.2 Bridge warning due

`POST /internal/events/bridge-warning-due`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "remaining_seconds": 60
}
```

### 10.3 Bridge cutoff due

`POST /internal/events/bridge-cutoff-due`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567"
}
```

### 10.4 Bridge ended

`POST /internal/events/bridge-ended`

**Request**
```json
{
  "call_session_id": "call_01JPK9VV71D3Q0N3G2P5R5B8D1",
  "phone_e164": "+972501234567",
  "ended_at": "2026-03-16T09:46:21.011Z",
  "reason": "star_exit"
}
```

### 10.5 Retry behavior

For all Core API events:

- retry on 5xx or timeout using exponential backoff
- maximum retry window: 60 seconds
- after max retries, log an error with masked phone number and keep cleaning up local resources
- do not block bridge teardown forever waiting on Core

---

## 11. OpenAI assistant behavior this service must enforce

At session creation, apply instructions equivalent to:

- Speak in clear, natural Hebrew by default
- Be concise and practical
- Ask one question at a time
- Avoid speaking too quickly
- Avoid slang that sounds foreign or internet-native
- If the caller asks in another language, switch appropriately
- Do not claim to be a rabbi, doctor, lawyer, therapist, or emergency responder
- For halachic, medical, legal, or urgent safety issues, politely advise the caller to consult a qualified human authority
- If the caller sounds emotionally distressed or in danger, respond calmly and encourage contacting a trusted person or emergency help immediately
- Never mention websites or internet-based steps unless explicitly asked
- Keep answers short enough for comfortable phone listening
- If the caller is silent for a while, gently invite them to continue

Output style rules:

- default language: Hebrew
- maximum uninterrupted assistant speech target: about 18 seconds
- no long verbal lists unless asked
- one response at a time

---

## 12. Audio and interruption behavior

### 12.1 Codec

Use `pcmu`.

### 12.2 Barge-in behavior

If caller speech starts while the assistant is speaking:

1. stop current assistant response output
2. clear queued assistant audio
3. keep conversation state aligned so the model does not assume the caller heard truncated speech
4. continue with the caller’s next turn

### 12.3 Star-exit behavior

If Telephony calls `/internal/bridge/end` because `*` was pressed:

- stop assistant audio immediately
- shut down the OpenAI session
- release the media bridge quickly
- emit `bridge-ended` with reason `star_exit`

### 12.4 Silence behavior

If caller is silent for 8 seconds after assistant audio fully stops:

- allow one short gentle re-engagement prompt
- do not loop endlessly
- if silence continues, just wait

---

## 13. Timer and cutoff behavior

This service does not decide whether a call is allowed. It only enforces timing values it receives.

### 13.1 Inputs

At session start, the service receives:

- `absolute_cutoff_epoch_ms`
- `warning_at_seconds`

### 13.2 Required timer loop

Run a lightweight loop every 500 ms or better.

Compute:

`remaining_seconds = floor((absolute_cutoff_epoch_ms - now_ms) / 1000)`

### 13.3 Warning logic

If:

- `warningSent == false`
- `remaining_seconds <= warning_at_seconds`
- `remaining_seconds > 0`

Then:

1. emit `bridge-warning-due` exactly once
2. set `warningSent = true`

### 13.4 Cutoff logic

If:

- `remaining_seconds <= 0`

Then:

1. emit `bridge-cutoff-due`
2. end the bridge locally with reason `time_expired`
3. emit final `bridge-ended` with reason `time_expired` if Telephony has not already ended it

### 13.5 Race rule

If Telephony already ended the session, the cutoff handler must avoid duplicate cleanup.

---

## 14. Failure handling

### 14.1 OpenAI session creation failure

- return `ok=false` from `/internal/bridge/start`
- do not emit `bridge-connected`

### 14.2 OpenAI disconnect during live session

- attempt one quick recovery if practical and under about 3 seconds
- if recovery fails, end bridge with reason `openai_error`
- emit `bridge-ended`

### 14.3 Media socket failure

- end bridge with reason `telephony_disconnect`

### 14.4 Core callback failure

Retry as specified above, but do not leak resources forever.

### 14.5 Process shutdown

On SIGTERM / SIGINT:

- stop accepting new starts
- end all live sessions cleanly
- emit `bridge-ended` for each if possible
- exit only after cleanup timeout or success

---

## 15. Security and privacy rules

This service must never store or log:

- payment data
- raw Authorization tokens
- full phone numbers in ordinary logs unless masked
- raw audio payload dumps in normal operation
- full transcripts by default

Allowed logging fields:

- `call_session_id`
- masked `phone_e164`
- bridge state
- timing metrics
- error code
- OpenAI session reference if non-sensitive

---

## 16. Configuration and environment variables

Provide `.env.example` with at least:

```env
PORT=8080
NODE_ENV=development

CORE_API_BASE_URL=https://core.internal
CORE_API_TOKEN=replace_me

OPENAI_API_KEY=replace_me
OPENAI_REALTIME_MODEL=replace_me

LOG_LEVEL=info
SESSION_SWEEP_INTERVAL_MS=30000
WARNING_TIMER_INTERVAL_MS=500
MASK_PHONE_LOGS=true
```

---

## 17. Required tests

### 17.1 Unit tests

- validation for start
- validation for end
- timer warning logic
- timer cutoff logic
- duplicate session rejection
- cleanup idempotency
- phone masking in logs

### 17.2 Integration tests

With mocked Core and mocked OpenAI:

- successful bridge start
- failed OpenAI session creation
- warning event emitted once
- cutoff event emitted at zero
- star exit tears session down immediately
- duplicate `end` does not leak
- bridge-ended always emitted on normal teardown

### 17.3 Load/stability tests

Provide a script or documented test plan for multiple simultaneous sessions ensuring:

- no leaked timers
- no memory growth after repeated cycles
- acceptable latency

---

## 18. Definition of done

This repository is complete only when:

1. Telephony can call `/internal/bridge/start` and receive a live connected bridge
2. Caller audio reaches OpenAI and assistant audio returns
3. Assistant behavior follows the Hebrew phone-appropriate rules above
4. Caller interruption works cleanly
5. Telephony can terminate instantly with `/internal/bridge/end`
6. Warning and cutoff events are emitted correctly to Core
7. No transcripts or payment data are stored by default
8. The repo contains all code, tests, containerization, and docs needed to deploy and run it
