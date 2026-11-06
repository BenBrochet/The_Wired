# The Wired

I WILL ADD THE CODE SOON

**A distributed AI agent operating system that orchestrates intelligent work across machines, models, and services.**

The Wired is a personal AI infrastructure platform — a 24/7 controller that accepts jobs via Telegram, API, CLI, or voice, routes them to local or cloud LLMs based on data sensitivity policies, manages enrolled worker machines, monitors system health, controls IoT devices, and recovers automatically from failures.

> For the full deep-dive — every API endpoint, internal design, database schema, and configuration reference — see **[TECHNICAL_REFERENCE.md](TECHNICAL_REFERENCE.md)**.

---

## Architecture

```
 YOU (phone / browser / CLI / voice)
      |
      | HTTPS via Tailscale Serve
      v
┌─────────────────────────────────────────────┐
│            lain-controller                   │
│            (FastAPI + SQLite)                 │
│                                               │
│  Job Queue ── Router ── Device Registry       │
│  Audit Log ── IoT Bridge ── Voice Controller  │
│  Emergency Stop ── Planner ── Dashboard       │
│  Secrets (age) ── Notifications (ntfy)        │
└──────────────────┬──────────────────────────┘
                   │  Tailscale (private mesh)
       ┌───────────┼───────────┐
       v           v           v
 ┌──────────┐ ┌──────────┐ ┌──────────┐
 │  Worker  │ │  Worker  │ │ Napoleon │
 │  (GPU)   │ │  (CPU)   │ │(Telegram)│
 │ Ollama   │ │ llama.cpp│ │ Bot API  │
 │ Podman   │ │ Whisper  │ └──────────┘
 └──────────┘ │ Piper TTS│
       |      └──────────┘
  ┌────┴────┐
  v         v
┌───────┐ ┌───────┐
│ESP32  │ │ESP32  │  IoT via MQTT
│Sensor │ │Light  │  (TLS + HMAC-signed)
└───────┘ └───────┘
```

| Component | Role | Stack |
|-----------|------|-------|
| **Controller** | Routes jobs, manages devices, serves dashboard, stores state | FastAPI, SQLite, 50+ modules |
| **Worker** | Claims and executes jobs — chat, code, research, merge | Python daemon, MCP tools, Podman sandbox |
| **CLI** | Human interface — REPL, device management, policy config | Terminal REPL with model selection |

---

## What It Does

**Job Orchestration** — Submit work via Telegram, API, CLI, or voice. Jobs are long-poll claimed by workers, executed with lease-based recovery, and automatically retried on failure. 12 job types: `chat`, `code`, `review`, `plan`, `research`, `summarise`, `study`, `voice`, `stt`, `tts`, `iot`, `classify`.

**Policy-Based Model Routing** — Jobs don't pick models; rules do. Each job has a data class that controls where it can run:

| Data Class | Where It Runs |
|------------|---------------|
| `sensitive` | Local models only |
| `private` | Local preferred, approved cloud providers |
| `public` | Any model, any provider |

Large jobs are sorted by throughput, small jobs by latency. Local models always preferred when tied.

**Security** — API keys encrypted at rest with age (XSalsa20-Poly1305). Tailscale Whois identity on every request. 15-minute enrollment tokens. DNS rebinding protection. HMAC-signed IoT commands. SHA-256 hash-chained audit log.

**IoT Control** — ESP32 devices over MQTT with TLS, per-device credentials, signed commands with replay protection, automation rules engine, and fail-safe on disconnect.

**Voice** — Wake word (openwakeword) + push-to-talk, speech-to-text (faster-whisper on GPU), text-to-speech (piper-tts on CPU), all routed through the same job system.

**Monitoring** — Per-device CPU, RAM, GPU memory/temp/utilisation tracking. Raw metrics for 24h, hourly rollups for 30 days. Jobs only land on workers with enough headroom.

**Emergency Stop** — Global kill switch. Instantly halts all job claiming, pauses running work, persists across restarts, admin-only release.

**Disaster Recovery** — 12 failure scenarios tested with proven recovery paths. Worker disconnect, model failure, DB corruption, container crash, disk full, unauthorised access, timeouts, cancellations, duplicates, backup restore, and emergency stop — all handled.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.10+ |
| API | FastAPI + Uvicorn |
| Database | SQLite (37 migrations, 38+ tables) |
| HTTP | httpx (async) |
| Tools | MCP (Model Context Protocol) |
| Encryption | pyrage (age) |
| Containers | Podman |
| Networking | Tailscale |
| Notifications | ntfy |
| MQTT | Mosquitto + paho-mqtt |
| Monitoring | Prometheus + Grafana |
| Testing | pytest + hypothesis (199 test files) |
| Firmware | C++ / PlatformIO (ESP32) |
| CI | GitHub Actions |

**LLM Providers** — model-agnostic via provider abstraction:
- Any OpenAI-compatible server (Ollama, llama.cpp, vLLM)
- Anthropic Claude
- Google Gemini

---

## How It Works

```
Submit ──→ Queued ──→ Claimed ──→ Running ──→ Completed
                        │                        │
                        │ (lease expires)         │ (or Failed)
                        └──→ Re-queued ──→ ...    v
                                              Artifacts uploaded
```

1. Job enters via API, Telegram, or voice
2. Planner converts intent into a structured job spec
3. Router evaluates policy rules, selects eligible models
4. Worker long-polls and claims the job (25s timeout)
5. Worker executes — the agent loop runs up to 20 tool hops with human-in-the-loop approval for writes
6. Results posted back, artifacts uploaded
7. If the worker dies, the lease expires and the sweeper re-queues

---

## CLI Example

```
$ lain
lain v0.1 · select a model

  1. mac/qwen3-coder:8b (ollama)
  2. mac/llama3.2:3b (ollama)

Model: 1
lain (qwen3-coder:8b) > research the latest news about AI agents
```

```bash
lain devices invite               # 15-min enrollment token
lain devices approve <id>         # Approve a pending worker
lain policy show                  # Display routing rules
lain secrets set provider/claude  # Store encrypted API key
lain-controller serve             # Start the controller
lain-controller selftest          # Database integrity check
lain-controller loadtest          # Benchmark throughput
lain-worker enrol --name gpu-box --kind workstation --token <token>
lain-worker run --jobs 4          # Start claiming jobs
```

---

## Deployment

8 Docker Compose services with optional profiles:

```bash
docker compose up -d controller ntfy                       # Core
docker compose --profile iot up -d                         # + IoT
docker compose --profile monitoring up -d                  # + Grafana/Prometheus
docker compose --profile iot --profile monitoring up -d    # Everything
```

Or bare metal via `deploy/setup.sh` — a 40-step automated install (BIOS, LUKS, Tailscale, systemd, age keys, migrations).

---

## Hardening

| Scenario | Recovery | Data Loss |
|----------|----------|-----------|
| Worker disconnects mid-job | Auto (lease + sweep) | None |
| Model endpoint fails | Immediate retry + backoff | None |
| Container crash | Job fails, worker continues | None |
| Database corruption | Selftest + backup restore | Since backup |
| Emergency stop | Instant stop, instant release | None |
| Duplicate submission | Idempotency key dedup | None |

All 12 scenarios documented in detail in [TECHNICAL_REFERENCE.md](TECHNICAL_REFERENCE.md).

---

## Development Phases

| Phase | Name | What Was Built |
|-------|------|----------------|
| OS0 | Audit | Hash-chained audit log |
| OS1 | EliteBook Setup | 40-step automated deployment |
| OS2 | Telegram | bot bridge + planner |
| OS3 | Workers | Multi-machine job claiming |
| OS4 | Emergency Stop | Global kill switch |
| OS5 | Multi-Provider | Policy-based model routing |
| OS6 | Code Agents | Code generation on branches |
| OS7 | Containers | Podman sandboxing |
| OS8 | Memory | Scoped memory + full-text search |
| OS9 | Monitoring | Device metrics + Prometheus/Grafana |
| OS10 | Hardening | 12 failure drills + load testing |

*For the complete technical reference — all API endpoints, agent loop internals, routing engine details, database schema, security model, configuration variables, and more — see **[TECHNICAL_REFERENCE.md](TECHNICAL_REFERENCE.md)**.*
