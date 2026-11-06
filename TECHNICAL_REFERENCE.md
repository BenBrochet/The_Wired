# Technical Reference

This document covers every detail of The Wired that didn't fit in the README:
API endpoints, agent internals, routing engine, worker daemon, database schema,
security model, IoT firmware, voice pipeline, and configuration reference.

---

## Why I Built This

I wanted a system where I could tell my AI to do things -- research, write code,
plan projects, control my house -- and have it figure out WHERE and HOW to run
the work. Not a chat wrapper. An operating system for AI agents. Heavily inspired by
ironman lain and other shows where agi is galore.

The problems I was solving:

- **Model routing:** Some jobs need to stay on local hardware (sensitive data).
  Others can go to Claude or Gemini for quality. The system should decide.
- **Multi-machine orchestration:** I have machines with different capabilities.
  Jobs should land on the right hardware automatically.
- **Reliability:** If a worker disconnects, the system recovers. If a model goes
  down, it fails over. If something goes wrong, I have an emergency stop.
- **Everything in one place:** Chat, code gen, web research, IoT, voice -- all
  through one unified job system with one audit trail.

---

## Full API Reference

The controller exposes a REST API on port 8700. All requests are authenticated
via Tailscale Whois identity.

### Health and Status

```
GET  /v1/health         -> {"schema": "lain.controller/1", "status": "ok"}
GET  /v1/whoami         -> Current authenticated principal info
GET  /v1/audit          -> Hash-chained audit log with verification
GET  /v1/stats          -> Lock waits and API latency metrics
GET  /v1/config/check   -> Configuration safety check
GET  /v1/health/move    -> Hardware move recommendation
```

### Device Management

```
POST /v1/devices/invites          -> Create 15-minute invite token
POST /v1/devices/enrol            -> Enrol with invite (creates pending device)
POST /v1/devices/{id}/approve     -> Approve with capabilities and roles
POST /v1/devices/{id}/heartbeat   -> Device checkin (every 15s)
PUT  /v1/devices/{id}/facts       -> Update device capabilities
PUT  /v1/devices/{id}/max-jobs    -> Set concurrency limit
PUT  /v1/devices/{id}/resources   -> Set min RAM/GPU requirements
POST /v1/devices/{id}/revoke      -> Revoke device access
GET  /v1/devices                  -> List all devices
```

### Job Management

```
POST /v1/jobs                     -> Submit a job (201 Created)
POST /v1/jobs/claim               -> Long-poll claim (25s default timeout)
POST /v1/jobs/{id}/heartbeat      -> Extend job lease
POST /v1/jobs/{id}/events         -> Log job events
POST /v1/jobs/{id}/complete       -> Mark completed
POST /v1/jobs/{id}/fail           -> Mark failed (with retry)
POST /v1/jobs/{id}/paused         -> Job paused notification
POST /v1/jobs/{id}/cancel         -> Cancel running job
```

### Artifacts

```
POST /v1/jobs/{id}/artifacts      -> Upload artifact (201 Created)
GET  /v1/jobs/{id}/artifacts      -> List job artifacts
GET  /v1/artifacts/{id}           -> Download artifact
```

### Routing and Policy

```
GET  /v1/routing/policies         -> Get model routing policy
PUT  /v1/routing/policies         -> Update routing rules
GET  /v1/routing/explain          -> Show which rule matches a job
```

### Emergency Control

```
GET  /v1/emergency                -> Get stop state
POST /v1/emergency/stop           -> Engage emergency stop (body: {"why": "..."})
POST /v1/emergency/release        -> Release emergency stop
```

### Containers

```
GET    /v1/containers/grants          -> List container operation grants
POST   /v1/containers/grants          -> Create grant (201 Created)
DELETE /v1/containers/grants/{id}     -> Revoke grant
POST   /v1/containers/{device}/ops   -> Execute container operation
```

### IoT Devices

```
POST   /v1/iot/devices               -> Register house device (201 Created)
GET    /v1/iot/devices               -> List devices
GET    /v1/iot/devices/{name}        -> Get device with latest readings
DELETE /v1/iot/devices/{name}        -> Deregister device
PUT    /v1/iot/devices/{name}/commands -> Enable/disable commands
POST   /v1/iot/events               -> Report device telemetry (from bridge)
GET    /v1/iot/polled               -> Poll for device commands
```

### IoT Commands

```
POST /v1/iot/commands                -> Send command to device (201 Created)
POST /v1/iot/commands/{id}/approve   -> Approve command
POST /v1/iot/commands/{id}/reject    -> Reject command
GET  /v1/iot/commands                -> List pending commands
GET  /v1/iot/commands/outbox         -> Device-facing command queue
```

### IoT Rules

```
POST   /v1/iot/rules                -> Create automation rule (201 Created)
GET    /v1/iot/rules                -> List rules
DELETE /v1/iot/rules/{id}           -> Delete rule
```

### Voice

```
POST /v1/voice/ask                   -> Speech-to-text + answer (WAV body)
```

Voice state controlled via settings table: wake, threshold, muted, enabled.

### Metrics

```
GET /v1/metrics/devices/{id}         -> Device history (raw or hourly rollups)
GET /v1/metrics/summary              -> Current metrics for all active devices
```

### Memory and Approvals

```
GET  /v1/memory                      -> Query scoped memory
GET  /v1/approvals                   -> Get pending approvals
POST /v1/approvals/{id}              -> Respond with verdict
```

---

## Agentic Loop Engine

File: `lain/agent/loop.py`

The turn engine drives all AI interactions. It sends the conversation to the
model, processes tool calls, and loops until the model produces a text response
or hits the hop limit.

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_TOOL_HOPS` | 20 | Prevents runaway tool loops |
| `TOOL_TIMEOUT` | 60s | Hard timeout per tool call |
| `MAX_TOOL_RESULT` | 64,000 | Truncates large tool outputs |
| `NEEDS_APPROVAL` | write_file, edit_file, run_command | Requires human approval |
| `FETCH_TOOLS` | fetch_url | Fetch operations |

### Key Functions

- `mcp_to_openai(tools)` -- Convert MCP tool definitions to OpenAI format for model consumption.
- `call_one_tool(session, call, tool_name, args, *, approver, sink)` -- Execute a single tool call. If the tool is in NEEDS_APPROVAL, prompt the user first. Runs via MCP session with 60s timeout.
- `run_turn(client, base, headers, model, tools, tool_to_session, messages, checkpoint, *, approver, sink)` -- One full assistant turn. Sends conversation, processes tool calls in a loop (up to MAX_TOOL_HOPS), returns when the model produces text.

### Turn Loop Process

1. Send conversation to model with available tools
2. If no tool_calls, assistant answered, return text
3. For each tool_call, check approval if in NEEDS_APPROVAL
4. Execute tool via MCP session with 60s timeout
5. Append tool result to conversation
6. Repeat (up to 20 hops)
7. If max hops exceeded, ask approver to continue

---

## Routing Policy Engine

File: `lain/controller/router.py`

Every job is matched against routing rules to determine which models can run it.

### Rule Structure

```json
{
  "name": "coding",
  "match": {
    "kind": ["code", "review"],
    "data_class": ["public", "private"],
    "needs": ["tools"]
  },
  "allow": ["trust:local", "claude/claude-*"],
  "enabled": true
}
```

### Match Fields

All optional -- missing matches anything.

| Field | Description |
|-------|-------------|
| `kind` | Job type: chat, code, review, plan, research, summarise, study, voice, stt, tts, iot, classify |
| `data_class` | Data sensitivity: public, private, sensitive |
| `needs` | Required capabilities (subset of job's requested capabilities) |

### Allow Selectors

| Selector | Meaning |
|----------|---------|
| `provider/model` | Specific model (supports * and ? globs) |
| `provider/*` | All models from a provider |
| `trust:local` | Any local model (Ollama, llama.cpp) |
| `trust:cloud` | Any cloud model (Claude, Gemini) |
| `*` | Anything |

### Model Preference

Large jobs (code, review, plan, research, summary, study, planning, or prompts >8,000 chars) are sorted by throughput (`gen_tps`).
Small jobs are sorted by latency (`p50_latency_ms`).
Local models are always preferred when tied.

### Default Routing Rules (9 built-in)

| # | Name | Match | Allow |
|---|------|-------|-------|
| 1 | image-analysis | needs vision | trust:local, trust:cloud |
| 2 | voice | kind=voice | trust:local only |
| 3 | speech-recog | kind=stt | trust:local, trust:cloud |
| 4 | iot-automation | kind=iot | trust:local only |
| 5 | classification | kind=classify | trust:local only |
| 6 | planning | kind=plan | trust:local, trust:cloud |
| 7 | coding | kind=code,review | trust:local, trust:cloud |
| 8 | research | kind=chat,research,summarise | trust:local, trust:cloud |
| 9 | anything-else | {} | trust:local only |

---

## Auto Kill Switch

File: `lain/controller/emergency.py`

A global kill switch stored in the settings table (key: `emergency_stop`).

### Stop Payload

```json
{
  "by": "username",
  "at": "2026-11-05T12:34:56Z",
  "why": "reason text",
  "paused": ["job_id_1", "job_id_2"]
}
```

### Effects

- All running jobs are paused immediately
- No new jobs can be claimed
- State persists across controller restarts (SQLite)
- Only authenticated admins can release (not Telegram)
- Push notification sent with priority=5, tag="octagonal_sign"
- Release resumes all paused jobs automatically

---

## Audit Log

File: `lain/controller/audit.py`

Append-only, hash-chained event log using SHA-256.

Genesis hash: `"0" * 64` (starting hash for the first row).

Each row hashes: prev_hash, timestamp, principal, action, resource, decision, detail.

### Functions

- `append(conn, principal, action, resource, decision, detail=None)` -- Transaction-isolated INSERT into audit_log.
- `verify(conn)` -- Walks the entire chain. Returns `(True, None)` or `(False, first_bad_id)` if any row is modified or deleted.
- `recent(conn, limit=100)` -- Last N rows with detail parsed from JSON.

### What Gets Logged

- Device enrollments, approvals, heartbeats, revocations
- Job claims, completions, cancellations, failures
- Emergency stops and releases
- Policy changes
- Container operations
- IoT commands

---

## Worker Daemon

File: `lain/worker/daemon.py`

Workers are autonomous processes that register with the controller and
compete for jobs.

### CLI Commands

```bash
lain-worker enrol --name <name> --kind workstation --token <invite>
lain-worker run [--jobs N] [--file device.json] [--serve-only]
```

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `HEARTBEAT_S` | 15 | Device checkin interval |
| `VERIFY_HOSTS_S` | 300 | Project verification interval |
| `CLAIM_WAIT_S` | 25 | Long-poll timeout |
| `MAX_BACKOFF_S` | 60 | Max retry backoff |
| `DRAIN_S` | 60 | Grace period on shutdown |

### Worker Lifecycle

1. Enrol with invite token, receive device.json
2. Start: report facts, verify project hosts, tidy worktrees
3. `work_forever()`: claim up to max_jobs, execute, report back
4. `heartbeat_forever()`: send load data every 15s
5. `verify_hosts_forever()`: check project checkouts every 300s
6. `drain(timeout=60)`: let running jobs finish before shutdown

### Load Reporting

| Metric | Source |
|--------|--------|
| `running_jobs` | Count of active jobs |
| `cpu` | `os.getloadavg()[0]` |
| `ram_free_mb` | `/proc/meminfo` MemAvailable |
| `gpus` | nvidia-smi query: index, name, memory.used, memory.total, temperature.gpu, utilization.gpu |

### Modes

- **Normal:** claim and execute jobs
- **Serve-only:** model host (heartbeats only, no job claiming)

### Job Runners

| Runner | Purpose |
|--------|---------|
| `chat.py` | Chat agent loop (provider API) |
| `code.py` | Code generation with linting and tests |
| `merge.py` | Auto-merge branches (git operations) |
| `research.py` | Web research with multi-turn loops |

### Sandbox Isolation

- **Worktrees:** isolated git branches per job
- **Containers:** Podman with per-device grants

---

## Metrics System

File: `lain/controller/metrics.py`

### Retention Policy

| Tier | Retention | Pruning |
|------|-----------|---------|
| Raw metrics | 24 hours | Hourly, batch 5000 rows |
| Hourly rollups | 30 days | Hourly, batch 5000 rows |

### Raw Metrics (device_metrics table)

```
device_id, at, cpu, ram_free_mb, ram_total_mb, gpu_free_mb, gpu_total_mb,
running_jobs, gpu_temp_c, gpu_util_pct
```

### Hourly Rollups (device_metrics_hourly table)

```
cpu_avg, ram_free_mb_min, ram_free_mb_avg, gpu_free_mb_min, gpu_free_mb_avg,
running_jobs_max, gpu_temp_c_max, gpu_util_pct_avg, samples
```

### Resource Placement

Jobs only land on workers with enough free RAM and GPU memory.
Resource floors are configurable per device via:

```
PUT /v1/devices/{id}/resources  -> {min_ram_mb, min_gpu_mb}
```

---

## Web Tools and Security

File: `lain/tools/web.py`

### MCP Tools

- `web_search(query, max_results=5)` -- DuckDuckGo search via ddgs
- `fetch_url(url)` -- Fetch with DNS rebinding protection

### DNS Rebinding Protection

1. Parse URL (http/https only)
2. Check robots.txt (unreachable = allowed)
3. Resolve hostname once to IP addresses
4. Verify ALL resolved IPs are public (not private/local)
5. Connect to the resolved IP with SNI for TLS
6. Stream body, stop at MAX_FETCH_BYTES

### Blocked IP Ranges

| Range | Reason |
|-------|--------|
| 10.0.0.0/8 | Private |
| 172.16.0.0/12 | Private |
| 192.168.0.0/16 | Private |
| 100.64.0.0/10 | Carrier-grade NAT / Tailscale |
| 127.0.0.0/8 | Loopback |
| 169.254.0.0/16 | Link-local |

### Limits

| Limit | Value |
|-------|-------|
| `MAX_REDIRECTS` | 5 |
| `MAX_FETCH_BYTES` | 2 MB |
| `MAX_CHARS` | 6,000 (returned text cap) |
| `MAX_ROBOTS_BYTES` | 512 KB |

---

## Device Enrollment

File: `lain/controller/devices.py`

### Constants

| Constant | Value |
|----------|-------|
| `INVITE_TTL` | 15 minutes |
| `HEARTBEAT_EVERY` | 15 seconds |
| `OFFLINE_AFTER` | 45 seconds (3 missed heartbeats) |

### Device Types

workstation, controller, phone, gateway, microcontroller, printer

### Facts Reported by Workers

```
os, arch, cpu_model, cpu_cores, ram_mb, gpus (list), storage (list),
endpoints (dict), hostname, tools (dict, max 20 keys)
```

### Enrollment Flow

1. Admin: `POST /v1/devices/invites` -- 15-min token
2. Device: `POST /v1/devices/enrol` with token -- pending device created
3. Admin: `POST /v1/devices/{id}/approve` with capabilities and roles
4. Device: can now heartbeat and claim jobs

---

## IoT System

Files: `lain/controller/iot.py`, `iot_commands.py`, `iot_rules.py`

### Telemetry

- Interval range: 30-3600 seconds
- Event kinds: tele (telemetry), ack (acknowledgement), dropped
- Sequence number: 0-based, increments per message

### Commands

- Signed with HMAC (LAIN_COMMAND_KEY stored in secrets.age)
- Nonce for idempotency
- Timestamp-based acceptance (prevents replay)

### Automation Rules

Trigger actions based on sensor readings.
Example: "if temperature > 30, turn on fan".
Rules evaluated on each telemetry event.

### ESP32 Firmware

File: `firmware/esp32/src/main.cpp`

- PubSubClient + WiFiClientSecure
- TLS to broker with house CA (never public CAs)
- Command signature verification with bootTime
- Sequence counter for telemetry ordering
- Actuators with safety limits (level control or digital)
- Fail-safe on broker disconnect (actuators to safe state)
- NTP clock sync required before accepting commands
- Configurable sensors/actuators from device config JSON

---

## Voice Pipeline

Files: `lain/controller/voice.py`, `lain/voice/service.py`, `lain/voice/stt_server.py`

### Pipeline

```
Microphone -> Wake Word / PTT -> Record -> Whisper (GPU) -> Transcription
-> Job submitted -> LLM response -> Piper TTS (CPU) -> Speaker
```

### Components

| Component | Technology |
|-----------|-----------|
| Wake word | openwakeword (configurable confidence threshold) |
| Push-to-talk | evdev (Linux input device) |
| STT | faster-whisper on GPU host |
| TTS | piper-tts on controller CPU (low latency) |

### Settings

Stored in settings table, key: `voice_state`.

| Setting | Default | Description |
|---------|---------|-------------|
| `wake` | false | Always-listening enabled |
| `threshold` | -- | Wake word confidence 0-1 |
| `muted` | false | Capture disabled |
| `enabled` | true | Overall on/off |

### Modes

muted, idle, push-to-talk, listening

### Concurrency

| Setting | Value |
|---------|-------|
| `CONCURRENT` | 2 max simultaneous voice calls |
| `WAIT_S` | 10s wait time before "too busy" |
| `VOICE_PRIORITY` | 45 (just below scheduled=50) |
| Max recording | 2 MB (~1 minute at 16kHz 16-bit mono) |

---

## Database Schema

SQLite with 37 forward-only migrations applied on startup.

### Key Tables

| Table | Purpose |
|-------|---------|
| `devices` | Registered workers with capabilities and roles |
| `device_invites` | Time-limited enrollment tokens |
| `device_metrics` | Raw resource snapshots (24h retention) |
| `device_metrics_hourly` | Rolled-up metrics (30-day retention) |
| `device_tools` | Per-device tool assignments |
| `jobs` | Job queue with status, kind, data class, model |
| `job_effects` | Idempotent one-time operations |
| `projects` | Registered project repositories |
| `routing_rules` | Policy-based model routing configuration |
| `model_registry` | Known models with provider and capabilities |
| `model_speed` | Throughput and latency benchmarks per model |
| `model_checks` | Health check results |
| `audit_log` | Hash-chained event log |
| `settings` | Key-value config (emergency stop, voice state) |
| `memory` | Scoped memory items with full-text search |
| `schedules` | Recurring job schedules |
| `plan_steps` | Goal/plan tracking |
| `iot_devices` | Registered ESP32 devices |
| `iot_readings` | Sensor telemetry |
| `iot_commands` | Signed command history |
| `iot_rules` | Automation triggers |
| `iot_hourly` | Rolled-up IoT metrics |
| `container_grants` | Per-device container operation permissions |
| `approvals` | Pending human-in-the-loop decisions |
| `usage` | Token usage tracking per model |
| `daily_caps` | Per-provider daily spending limits |
| `napoleon` | Telegram bridge state |

### Migration Files (0001-0037)

| # | File | # | File |
|---|------|---|------|
| 0001 | registry.sql | 0020 | goal_tracking.sql |
| 0002 | providers_models.sql | 0021 | voice_wakes.sql |
| 0003 | device_invites.sql | 0022 | iot.sql |
| 0004 | jobs.sql | 0023 | iot_actuators.sql |
| 0005 | projects.sql | 0024 | iot_commands.sql |
| 0006 | routing.sql | 0025 | iot_hourly.sql |
| 0007 | model_checks.sql | 0026 | iot_rules.sql |
| 0008 | cloud_optins.sql | 0027 | iot_commands.sql |
| 0009 | settings.sql | 0028 | iot_address.sql |
| 0010 | usage.sql | 0029 | iot_prints.sql |
| 0011 | daily_caps.sql | 0030 | model_speed.sql |
| 0012 | memory.sql | 0031 | device_job_limit.sql |
| 0013 | memory_fts_triggers.sql | 0032 | napoleon.sql |
| 0014 | memory_fts_stemming.sql | 0033 | approvals.sql |
| 0015 | project_personal_memory | 0034 | device_tools.sql |
| 0016 | schedules.sql | 0035 | job_effects.sql |
| 0017 | plan_steps.sql | 0036 | container_grants.sql |
| 0018 | project_job_limit.sql | 0037 | device_metrics.sql |
| 0019 | pause_requested.sql | | |

---

## Deployment

### Docker Compose Stack

File: `deploy/compose.yaml`

| Service | Image | Port | Memory | Profile |
|---------|-------|------|--------|---------|
| controller | lain-controller:0.1 | 127.0.0.1:8700 | 600 MB | core |
| ntfy | binwiederhier/ntfy:v2.11.0 | 127.0.0.1:2586 | 128 MB | core |
| mosquitto | eclipse-mosquitto:2.0.22 | LAN_IP:8883 | 128 MB | iot |
| mqtt-bridge | lain-controller:0.1 (lain-iot-bridge) | -- | 128 MB | iot |
| prometheus | prom/prometheus:v2.55.1 | 127.0.0.1:9090 | 512 MB | monitoring |
| node-exporter | prom/node-exporter:v1.8.2 | 127.0.0.1:9100 | 64 MB | monitoring |
| grafana | grafana/grafana-oss:11.3.0 | 127.0.0.1:3000 | 256 MB | monitoring |

Prometheus retention: 30 days, max 5 GB.

### Entry Points

From `pyproject.toml`:

```
lain            = lain.cli.repl:run
lain-controller = lain.controller.main:main
lain-worker     = lain.worker.daemon:main
lain-voice      = lain.voice.service:main
lain-stt        = lain.voice.stt_server:main
lain-iot-bridge = lain.iot.mqtt_bridge:main
```

### Bare Metal

File: `deploy/setup.sh`

40-step automated installation covering:

- BIOS configuration and LUKS full-disk encryption
- Tailscale installation and network join
- age key generation for secrets encryption
- systemd service setup (lain-stack.service)
- Database initialization and migration
- Health monitoring with automatic restart

---

## Security Model

| Layer | Implementation |
|-------|---------------|
| Network | Tailscale mesh -- no public IPs, no port forwarding |
| Identity | Tailscale Whois on every request -- identity tied to machine |
| Secrets | age encryption (XSalsa20-Poly1305) -- keys never in DB or logs |
| Enrollment | 15-minute invite tokens, admin approval required |
| Data Policy | Sensitive data never leaves local hardware |
| Web Fetching | DNS rebinding protection, private IP blocking, robots.txt |
| IoT | TLS + HMAC-signed commands with nonce replay protection |
| Audit | SHA-256 hash-chained log -- tampering detectable |
| Containers | Grant-based Podman sandboxing per device |
| Tools | Human-in-the-loop approval for write/edit/run operations |

---

## Hardening and Disaster Recovery -- All 12 Scenarios

| # | Scenario | Recovery | Data Loss |
|---|----------|----------|-----------|
| 1 | Worker disconnects mid-job | lease_s + sweep_s (auto) | None |
| 2 | Model endpoint fails | Immediate retry + backoff | None |
| 3 | Telegram delivery fails | Instant (non-blocking) | Message text |
| 4 | Container crash | Job fails, worker continues | None |
| 5 | Database corruption | selftest + backup restore | Since backup |
| 6 | Disk space low | Health alert (5 min) | None |
| 7 | Unauthorised request | 401/403, audited | None |
| 8 | Agent timeout | Same as worker disconnect | None |
| 9 | Job cancelled mid-run | 5s or less (heartbeat interval) | Partial work |
| 10 | Duplicate submission | idempotency_key dedup | None |
| 11 | Backup restore | Selftest passes, audit OK | Since backup |
| 12 | Emergency stop | Instant stop + release | None |

---

## Testing

199 test files covering the full system.

### Test Categories

- **Gate tests:** test_os0_gate.py through test_os10_gate.py (one per phase)
- **Plan tests:** test_p*_gate.py (underlying plan-level validation)
- **Feature tests:** routing, policy, memory, voice, iot, contracts, etc.

### Test Infrastructure

- Fake model server -- no real LLM needed
- Temporary ~/.config/lain/ per test run -- no real config touched
- Hypothesis property-based tests for state transitions
- CI runs on Python 3.10 and 3.15 (Ubuntu, GitHub Actions)

### CI Pipeline

File: `.github/workflows/ci.yml`

1. uv sync with uv.lock (exact versions)
2. Ruff linting check
3. pytest -q
4. Docker image build: `docker build -f deploy/Dockerfile -t lain-controller:ci`
5. Compose validation: `docker compose config -q`

### Load Test Results

10 workers, 1,000 jobs -- 104.8 jobs/sec, p95 claim latency 113.6ms.

### Dependencies

| Group | Packages |
|-------|----------|
| Core | httpx, mcp<2, ddgs |
| Server | fastapi, uvicorn, pyrage, jinja2 |
| Voice | sounddevice, evdev, piper-tts, openwakeword (Linux only) |
| STT | faster-whisper, python-multipart (Linux only) |
| IoT | paho-mqtt>=2.1, websockets>=16 |
| Cloud | anthropic |
| Dev | pytest, pytest-asyncio, hypothesis, ruff |

---

## Configuration Reference

All configuration via environment variables -- no hardcoded values.

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_DB` | /srv/lain/lain.db | SQLite database path |
| `LAIN_CONTROLLER_HOST` | 127.0.0.1 | Bind address |
| `LAIN_CONTROLLER_PORT` | 8700 | API port |
| `LAIN_ADMINS` | (required) | Comma-separated admin logins |
| `LAIN_AGE_KEY_FILE` | /run/secrets/age_key | Age decryption key path |
| `LAIN_SECRETS_FILE` | /srv/lain/secrets.age | Encrypted secrets file |
| `LAIN_ARTIFACTS_DIR` | /srv/lain/artifacts | Uploaded artifact storage |

### Scheduling and Timing

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_LEASE_S` | 60 | Job claim lease duration |
| `LAIN_BACKOFF_BASE_S` | 30 | Retry backoff base |
| `LAIN_SWEEP_S` | 5 | Reaper/scheduler cycle |
| `LAIN_MODEL_CHECK_S` | 300 | Model health check interval |
| `LAIN_TZ` | UTC | Timezone for schedules |

### Notifications

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_NTFY_URL` | http://127.0.0.1:2586 | ntfy server |
| `LAIN_NTFY_TOPIC` | lain | Notification topic |
| `LAIN_PUBLIC_URL` | https://<tailnet>:443 | Dashboard URL for links |

### IoT

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_MQTT_HOST` | (optional) | MQTT broker address |
| `LAIN_MQTT_PORT` | 8883 | MQTT port |
| `LAIN_LAN_IP` | (optional) | EliteBook LAN IP |

### Telegram

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_NAPOLEON_URL` | http://127.0.0.1:8000 | Napoleon API endpoint |
| `LAIN_NAPOLEON_ROUTE` | Lain/Lain | Napoleon domain/project |

### Tailscale

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_REQUIRE_WHOIS` | 1 | Require Tailscale identity |
| `LAIN_TAILSCALE_SOCKET` | /var/run/tailscale/tailscaled.sock | Socket path |

### Agent

| Variable | Default | Description |
|----------|---------|-------------|
| `LAIN_CONTEXT_CHARS` | 96000 | Context window budget |
| `LAIN_APPROVE_FETCH` | 0 | Require approval for fetch_url |
| `LAIN_AUTO_APPROVE` | 0 | Skip all tool approvals |
| `LAIN_STREAM` | 1 | Stream model output |
| `LAIN_LOG` | WARNING | MCP server log level |

---

## Development Phases

The system was built across 10 phases, each adding a major capability:

| Phase | Name | What Was Built |
|-------|------|----------------|
| OS0 | Audit | Hash-chained audit log, verification, API |
| OS1 | EliteBook Setup | 40-step automated deployment, systemd, LUKS |
| OS2 | Telegram | Napoleon bot bridge, intent-to-job planner |
| OS3 | Workers | Device enrollment, heartbeats, multi-machine claiming |
| OS4 | Emergency Stop | Global kill switch, pause/resume, admin-only release |
| OS5 | Multi-Provider | Claude, Gemini, OpenAI-compat routing with policies |
| OS6 | Code Agents | Code generation on branches, linting, test runners |
| OS7 | Containers | Podman sandboxing, per-device grants, isolation tests |
| OS8 | Memory | Scoped memory with full-text search, project context |
| OS9 | Monitoring | Device metrics, resource placement, Prometheus/Grafana |
| OS10 | Hardening | 12 failure drills, disaster recovery, load testing |

