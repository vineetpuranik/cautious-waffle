# GrokCast – Real‑Time AI Voice Conversation System

A production‑grade, low‑latency, end‑to‑end voice assistant that streams microphone audio from a browser to backend services for speech recognition (ASR), LLM reasoning, and text‑to‑speech (TTS) synthesis, returning audio in near real time.

This project is designed to exercise deep skills in:

* Low‑latency systems and protocols (WebRTC, WebSocket)
* High‑performance media pipelines (PCM/Opus, buffering, resampling, VAD)
* Concurrency and reliability (Rust async I/O, Python inference workers)
* AI integration (ASR, LLM, TTS) and streaming inference
* Observability, performance tuning, and production deployment

---

## 1) Why this project

GrokCast mirrors the problems you’d solve building an AI voice product at scale: ultra‑low latency, robust media quality across devices/networks, and end‑to‑end reliability from the mic to the synthesized response.

**Success criteria**

* E2E median (p50) mic‑to‑ear latency ≤ **300 ms** for short round‑trips (e.g., echo, hotword) and ≤ **600–900 ms** for ASR→LLM→TTS responses.
* High audio quality with graceful degradation on poor networks (adaptive bitrate, jitter buffering, PLC).
* Horizontal scaling for thousands of concurrent sessions.
* Full tracing/metrics across the pipeline.

---

## 2) High‑level architecture

```
[ Web Client ]                             [ Backend ]
Mic ▶ getUserMedia ▶ WebRTC/WS  ─────────►  Rust Gateway (Tokio/Axum)
  ▲                                         ├─ Session mgr (auth, QoS, ABR)
  │                                         ├─ Opus/PCM framing, jitter ctrl
  │                                         └─ gRPC/WS streams to Python ASR/LLM/TTS
  │
Playback ◄──────── WebRTC/WS ◄──────────  Python Inference (FastAPI/uvicorn)
                                            ├─ ASR (e.g., Faster‑Whisper)
                                            ├─ LLM (streaming; function calls)
                                            └─ TTS (e.g., Piper/Coqui)

Optional infra: Redis/Kafka (events, backpressure), Postgres (sessions/logs),
TURN/STUN for NAT traversal, Prometheus/Grafana/Tempo for obs.
```

### Data flow (happy path)

1. Browser captures audio frames (16 kHz or 48 kHz PCM) → encodes as Opus or raw PCM.
2. Client streams frames to **Rust Gateway** via **WebRTC media** or **WebSocket** (binary frames).
3. Gateway demuxes frames, manages jitter buffer, resamples if needed, and forwards chunks to **ASR**.
4. **ASR** streams partial transcripts; once intent is clear, system triggers **LLM**.
5. **LLM** streams tokens; gateway can start **TTS** on partial text (barge‑in aware) to minimize latency.
6. TTS emits audio chunks (Opus/PCM) → Gateway returns to browser for immediate playback.

---

## 3) Latency budget (target)

| Stage                       |  Target (ms) | Notes                                               |
| --------------------------- | -----------: | --------------------------------------------------- |
| Capture + encode (client)   |        10–30 | 10–20 ms frames; avoid large jitter buffers.        |
| Network uplink              |        10–50 | Depends on RTT; ABR and silence suppression help.   |
| Gateway ingress + buffering |         5–15 | Keep queues shallow; backpressure if needed.        |
| ASR partial                 |       60–150 | Faster‑Whisper (GPU) or server‑side VAD + chunking. |
| LLM first tokens            |       80–200 | Stream tokens; early TTS on chunks.                 |
| TTS first audio             |       60–200 | Prefer streaming TTS with low tail latency.         |
| Downlink + playback         |        10–40 | Small audio buffers; PLC enabled.                   |
| **Total p50**               | **~240–500** | Simple prompts; dependent on model sizes.           |

---

## 4) Components & responsibilities

### Web Client (TypeScript)

* Capture mic via `getUserMedia`.
* Send audio using **WebRTC** (Opus) or **WebSocket** (PCM/Opus frames).
* Jitter buffer and playback for response audio; barge‑in handling.
* Adaptive bitrate, DTX (silence suppression), packet loss concealment.
* UX: push‑to‑talk and continuous modes, partial transcript display.

### Rust Gateway (Tokio + Axum)

* **Transport termination**: WebRTC SFU‑lite or WS server.
* **Session management**: auth (JWT), rate‑limits, prioritization.
* **Media handling**: Opus decode/encode (optional), resampling, framing.
* **Flow control**: backpressure, out‑of‑order handling, reorder/write‑combining.
* **Routing**: stream to Python ASR/LLM/TTS via gRPC/WS; multiplex results.
* **Observability**: OpenTelemetry spans per stage; RED metrics (rate, errors, duration).

### Python Inference (FastAPI/uvicorn + model runtimes)

* **ASR**: streaming inference; partials + final.
* **LLM**: streaming text; function‑calling hooks (tool use) for actions.
* **TTS**: low‑latency voice streaming; voice presets and SSML.
* **Batching/parallelism**: micro‑batches with deadlines; GPU/CPU mix.
* **Safety**: content filters, prompt guards.

### Optional infra

* **Redis/Kafka** for async events, retries, and buffering.
* **Postgres** for session logs, metrics, transcripts (PII‑aware policies).
* **TURN/STUN** for WebRTC NAT traversal.
* **Prometheus + Grafana + Tempo/Jaeger** for metrics & traces.

---

## 5) Protocols & framing

### WebSocket binary frame (PCM example)

```
// 20 ms @ 16 kHz mono PCM16 = 320 samples = 640 bytes
struct AudioFrame {
  u32 session_id;
  u64 monotonic_ts_ns;   // capture time
  u16 seq;               // wraparound ok
  u8  codec;             // 0 = PCM16, 1 = Opus
  u8  flags;             // bit0: DTX, bit1: VAD_speech
  u16 payload_len;
  u8  payload[payload_len];
}
```

**WebRTC option:** prefer Opus RTP with SRTP security; leverage built‑in jitter/PLC.

---

## 6) Audio pipeline details

* **Sample rates**: standardize to 16 kHz (ASR‑friendly) or 24/48 kHz (playback quality). Use Rust `rubato` for resampling.
* **Codecs**: Opus for network; PCM internally for model I/O to avoid extra latency.
* **VAD/DTX**: client‑side VAD to avoid uploading silence; server confirms speech segment boundaries.
* **Jitter/PLC**: small jitter buffer (~40–80 ms) with packet loss concealment.
* **Barge‑in**: detect user speech while TTS is playing; attenuate or pause TTS.

---

## 7) Reliability & scaling

* **Stateless gateway** with sticky sessions (hash on session_id) for cache locality.
* **Graceful degradation**: drop to lower bitrate, reduce TTS prosody, or switch to partial‑text UI.
* **Backpressure**: bounded queues; if ASR/LLM lag, signal client to slow send.
* **Hot reload**: drain connections and rotate workers.
* **Multi‑region**: anycast or GSLB; co‑locate gateway & inference to minimize RTT.

---

## 8) Security & privacy

* TLS everywhere; SRTP for WebRTC.
* JWT‑based auth; short‑lived tokens.
* PII handling for transcripts; encryption at rest; data retention policies.
* Abuse protection: per‑IP/session rate limits, captcha for signup.

---

## 9) Observability

* **Tracing**: spans for capture→ingress→ASR→LLM→TTS→egress with session_id and seq tags.
* **Metrics**: p50/p90 latency per stage, packet loss %, jitter, SNR, CPU/GPU util.
* **Logging**: structured JSON, correlation IDs.
* **Dashboards**: Latency waterfall, ABR states, error maps, capacity.

---

## 10) Testing strategy

* **Load gen**: headless client that replays curated WAVs at scale.
* **E2E latency test**: embed timestamps in audio; measure round‑trip.
* **Chaos**: packet loss/insertion/reordering; network throttling.
* **Audio quality**: PESQ/STOI on reference clips; crowd tests.
* **Unit/integration** across Rust gateway (framing, jitter), Python models, and client.

---

## 11) Performance & SLOs

* **SLO**: p50 E2E ≤ 600 ms; p95 ≤ 1.2 s for short interactions.
* **Gateway**: ≥ 10k concurrent WS connections per node; zero‑copy where possible.
* **ASR**: partials within 120 ms; final within 400 ms for short utterances.
* **TTS**: first audio ≤ 200 ms; >1.0x realtime thereafter.

---

## 12) Tech stack

* **Client**: TypeScript, WebRTC APIs, AudioWorklets, WebAudio.
* **Gateway**: Rust (Tokio, Axum, tungstenite/warp), opentelemetry, prost/tonic for gRPC.
* **Inference**: Python (FastAPI), Faster‑Whisper, vLLM/LLM API, Piper/Coqui TTS.
* **Infra**: Docker/Compose, Redis/Kafka, Postgres, Prometheus/Grafana/Tempo.

---

## 13) Repository layout

```
/grokcast
  ├─ client/              # Web UI + signaling
  ├─ gateway/             # Rust gateway (Axum/Tokio)
  ├─ inference/           # Python ASR/LLM/TTS services
  ├─ proto/               # Protobuf/IDLs (gRPC + WS framing docs)
  ├─ ops/                 # Docker, k8s manifests, grafana dashboards
  ├─ tools/               # Load generator, trace analyzers
  └─ README.md
```

---

## 14) Local development (quick start)

**Prereqs**: Rust toolchain, Python 3.11, Node 20, Docker.

```bash
# 1) Build gateway
cd gateway && cargo run

# 2) Start inference services (ASR/LLM/TTS)
cd ../inference && uvicorn app:app --host 0.0.0.0 --port 8001

# 3) Run client (dev server)
cd ../client && npm i && npm run dev
```

Configure endpoints via `.env` in each package (see `client/.env.example`, `gateway/.env`, `inference/.env`).

---

## 15) API reference (sketch)

### WebSocket routes (gateway)

* `wss://<host>/v1/stream` – bidirectional audio+control stream.

**Client → Server control messages (JSON):**

```json
{ "type": "start", "codec": "opus", "sr": 48000, "bitrate": 24000 }
{ "type": "vad", "speech": true, "seq": 1024 }
{ "type": "end" }
```

**Server → Client control messages (JSON):**

```json
{ "type": "partial_transcript", "text": "hello wor", "ts": 123456789 }
{ "type": "final_transcript", "text": "hello world", "ts": 123556789 }
{ "type": "tts_state", "status": "speaking" }
```

**Audio frames** use binary messages with `AudioFrame` header (see §5).

---

## 16) Deployment

* **Docker Compose** for local multi‑service orchestration.
* **Kubernetes** (optional): Gateway `Deployment` with `PodDisruptionBudget`, HPA on CPU and active sessions; Inference with GPU nodes; Redis/Kafka as managed services.
* **TURN/STUN** (coturn) for WebRTC.
* **CI/CD**: lint, tests, load smoke (5 min), build, push images, progressive rollout.

---

# Roadmap (step‑by‑step to a complete app)

The roadmap is split into milestones. Each milestone is shippable and builds toward production readiness. Estimated durations assume part‑time evenings/weekends; compress if full‑time.

## Milestone 0 – Scaffolding (1–2 days)

* Create monorepo layout (client/gateway/inference/tools/ops).
* Add precommit hooks, linters (rustfmt, clippy, ruff), basic CI.
* Define `.env` contracts; write initial README and contribution guide.

**Exit criteria:** repo builds; hello‑world servers run.

## Milestone 1 – Transport baseline (3–5 days)

* Implement **WebSocket** audio echo path:

  * Client captures mic, 20 ms PCM16 frames → WS → Gateway.
  * Gateway echoes frames back → client plays out.
* Add sequence numbers, timestamps, and a tiny jitter buffer.
* Implement minimal metrics (frames/sec, RTT, packet loss est.).

**Exit criteria:** p50 echo round‑trip ≤ 120 ms on LAN.

## Milestone 2 – ASR streaming (3–7 days)

* Add ASR service (Faster‑Whisper). Stream partials + finals.
* Gateway fans out audio to ASR; client shows transcripts live.
* Introduce VAD and DTX (client) to reduce upload in silence.

**Exit criteria:** partials within 150 ms; finals coherent.

## Milestone 3 – LLM integration (3–5 days)

* Stream transcripts into LLM; return streaming text responses.
* Define conversation/session context; simple tool‑use hook.
* Display token stream in client.

**Exit criteria:** first tokens < 200 ms after ASR final (or partial gating).

## Milestone 4 – TTS streaming (4–7 days)

* Integrate low‑latency TTS; stream Opus back to client.
* Barge‑in: detect user speech, attenuate/pause TTS.
* Add output jitter buffer and PLC.

**Exit criteria:** first audio chunk < 200 ms after first LLM tokens.

## Milestone 5 – Performance pass (4–10 days)

* Latency budgeting: measure each stage; Grafana dashboards.
* Adaptive bitrate & network resilience tests (loss/latency).
* Backpressure semantics; bounded queues with metrics.

**Exit criteria:** meet SLOs (p50 ≤ 600 ms; p95 ≤ 1.2 s).

## Milestone 6 – Reliability & scaling (5–10 days)

* Horizontal scale gateway; sticky sessions; health checks.
* Add Redis/Kafka for decoupling (optional, feature‑flagged).
* Graceful deploys; connection draining; retries with idempotency keys.

**Exit criteria:** survive rolling restarts without user impact.

## Milestone 7 – Security & privacy (3–5 days)

* JWT auth; short‑lived tokens; TLS termination.
* PII policies for transcripts; redaction toggles; retention.
* Abuse controls: per‑IP/session limits; audit logs.

**Exit criteria:** baseline security review checklist passes.

## Milestone 8 – Testing & QA (3–7 days)

* Load generator tool (tools/loadgen) to replay WAVs concurrently.
* E2E latency harness with in‑audio timestamps.
* Chaos suite: packet loss/reorder, jitter, bandwidth throttling.

**Exit criteria:** automated nightly runs with thresholds.

## Milestone 9 – Packaging & deploy (3–5 days)

* Docker images; Compose for local; Helm chart for k8s.
* TURN/STUN configuration for WebRTC mode.
* CI/CD pipeline with progressive rollout.

**Exit criteria:** one‑click deploy to a small cloud cluster.

## Stretch goals (nice‑to‑have)

* Multimodal capture (image/video) with synchronized streams.
* On‑device VAD/keyword spotting (WASM) to reduce server load.
* Multi‑speaker diarization; better barge‑in policies.
* Voice cloning and per‑user prosody control.

---

## Contributing

PRs are welcome. Please open an issue for major changes. Follow the code of conduct and ensure all CI checks pass.

## License

Choose a permissive license (e.g., Apache‑2.0 or MIT) suitable for open collaboration.
