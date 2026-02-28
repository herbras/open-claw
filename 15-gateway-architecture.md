# OpenClaw: Gateway Architecture & Agent Runtime

> Arsitektur Gateway, agent loop, system prompt construction, dan operational reference.

---

## Konsep Dasar

| Istilah | Penjelasan |
|---------|-----------|
| **Gateway** | Process utama yang handle semua — routing, channel connections, agent runs |
| **Agent Runtime** | Runtime yang menjalankan agent (inference, tool execution, reply) |
| **Agent Loop** | Satu siklus lengkap: pesan masuk → context → LLM → tools → reply |
| **System Prompt** | Prompt yang dibangun dari SOUL.md, skills, bootstrap, dan overrides |

---

## Gateway — Single Process, Banyak Peran

Gateway adalah **satu process** yang handle segalanya:

```
Gateway (ws://0.0.0.0:18789)
├── Channel Connections
│   ├── WhatsApp (via Baileys)
│   ├── Telegram (via grammY)
│   ├── Discord, Slack, Signal, iMessage
│   └── WebChat
├── WebSocket API (typed, JSON)
│   ├── Control clients (CLI, macOS app, web UI)
│   └── Nodes (macOS, iOS, Android, headless)
├── HTTP Server
│   ├── OpenAI-compatible API
│   ├── Canvas host
│   ├── Webhooks
│   └── Dashboard
└── Agent Runtime
    ├── Agent runs (queue + concurrency)
    ├── Tool execution
    └── Session management
```

### Runtime Model

- **Single multiplexed port** — WS + HTTP pakai port yang sama (default 18789)
- **Default bind**: `loopback` (127.0.0.1 only)
- **Auth required** — token atau password (wajib untuk non-loopback bind)
- **Satu Gateway per host** — hanya satu yang boleh buka WhatsApp session

### Port & Bind Precedence

| Setting | Urutan Resolusi |
|---------|-----------------|
| Port | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789` |
| Bind | CLI/override → `gateway.bind` → `loopback` |

---

## Agent Loop — Dari Pesan ke Reply

Ini yang terjadi setiap kali pesan masuk:

### 1. Intake

```
Pesan masuk (WhatsApp/Telegram/dll)
    ↓
Gateway validate params
    ↓
Resolve session (sessionKey/sessionId)
    ↓
Return { runId, acceptedAt } immediately
```

### 2. Queueing

- Runs **serialized per session** (1 run at a time per session)
- Optional global queue
- Channel bisa pilih queue mode: `collect`, `steer`, `followup`

### 3. Session & Workspace Preparation

```
Resolve workspace
    ↓
Load skills (atau reuse snapshot)
    ↓
Inject bootstrap/context files
    ↓
Acquire session write lock
    ↓
Open SessionManager
```

### 4. Prompt Assembly

```
Base prompt (OpenClaw)
    + Skills prompt
    + Bootstrap context (SOUL.md, TOOLS.md, BOOTSTRAP.md)
    + Per-run overrides
    = System Prompt → kirim ke LLM
```

### 5. LLM Inference + Tool Execution

```
System Prompt + History + User Message → LLM
    ↓
LLM response (bisa: text, tool calls, atau keduanya)
    ↓
Kalau tool call → execute tool → kirim result balik ke LLM → loop
    ↓
Kalau text → stream ke user
```

### 6. Reply & Persist

```
Final reply → stream ke chat channel
    ↓
Persist ke session JSONL transcript
    ↓
Update session metadata
    ↓
Emit lifecycle end event
```

### Timeouts

| Timeout | Default | Config |
|---------|---------|--------|
| Agent runtime | 600s (10 menit) | `agents.defaults.timeoutSeconds` |
| `agent.wait` | 30s | `timeoutMs` param |

---

## System Prompt Construction

System prompt dibangun berlapis:

```
1. OpenClaw base prompt (internal)
2. + Skills prompt (dari loaded skills)
3. + Bootstrap files:
     - SOUL.md (personality, rules)
     - TOOLS.md (tool documentation)
     - BOOTSTRAP.md (initial context)
     - IDENTITY.md (agent identity)
     - USER.md (user info)
     - HEARTBEAT.md (heartbeat config)
     - AGENTS.md (multi-agent info)
4. + Per-run overrides (model, thinking, etc.)
5. + Injected context (hooks, plugins)
```

### bootstrapMaxChars

Total karakter dari bootstrap files yang di-load dibatasi:

```json
{
  "agents": {
    "defaults": {
      "bootstrapMaxChars": 40000
    }
  }
}
```

Kalau total melebihi limit, files di-truncate. Keep SOUL.md concise!

### Context Inspection

Kirim di chat untuk inspect:
- `/context list` — lihat apa yang ada di system prompt
- `/context detail` — detail ukuran tiap context contributor

---

## Hot Reload — Config Changes Tanpa Restart

Gateway bisa auto-detect perubahan `openclaw.json`:

| Mode | Perilaku |
|------|----------|
| `off` | Tidak ada reload |
| `hot` | Apply perubahan yang "safe" saja |
| `restart` | Restart kalau ada perubahan yang butuh restart |
| `hybrid` (default) | Hot-apply kalau safe, restart kalau perlu |

```json
{
  "gateway": {
    "reload": {
      "mode": "hybrid"
    }
  }
}
```

**GOTCHA dari pengalaman kita:** Hot reload bisa bikin bingung. Kalau config baru invalid, gateway tetap jalan dengan config lama/fallback. Baru ketahuan error setelah manual restart. Lihat `03-whatsapp-group-setup.md` gotcha #4.

---

## Streaming & Chunking

Agent reply di-stream secara real-time:

- **Assistant deltas** — streamed dari LLM, dikirim sebagai partial messages
- **Block streaming** — emit partial reply di `text_end` atau `message_end`
- **Tool events** — start/update/end streamed ke client

### Chunking di WhatsApp

WhatsApp punya batas panjang pesan. OpenClaw otomatis chunk reply panjang jadi beberapa pesan.

---

## Retry Policy

Kalau LLM call gagal (rate limit, timeout, dll):

```json
{
  "agents": {
    "defaults": {
      "retry": {
        "maxRetries": 3,
        "backoff": "exponential"
      }
    }
  }
}
```

Auto-compaction bisa trigger retry — kalau context window penuh, compact dulu lalu retry.

---

## OAuth

OpenClaw support OAuth untuk authenticate ke services:

```json
{
  "oauth": {
    "providers": {
      "google": {
        "clientId": "xxx",
        "clientSecret": "xxx",
        "scopes": ["email", "profile"]
      }
    }
  }
}
```

Use case: login ke Google, GitHub, dll lewat agent.

---

## Operational Commands

### Day-to-Day

```bash
# Start gateway
ssh my-vps "cd ~/openclaw && docker compose up -d openclaw-gateway"

# Status
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js status"

# Health check
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js health"

# Doctor (audit & repair)
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js doctor"

# Logs (live)
ssh my-vps "cd ~/openclaw && docker compose logs -f openclaw-gateway"

# Channel status
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js channels status --probe"
```

### Config Management

```bash
# Get current config
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js gateway config.get"

# Apply config changes (validate + write + restart)
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js gateway config.apply"

# Patch config (merge partial update)
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js gateway config.patch"
```

### Restart & Update

Agent punya `gateway` tool yang bisa restart/update:

| Action | Fungsi |
|--------|--------|
| `restart` | Restart gateway in-place (SIGUSR1) |
| `config.get` | Get current config |
| `config.schema` | Get config JSON schema |
| `config.apply` | Validate + write config + restart |
| `config.patch` | Merge partial update + restart |
| `update.run` | Run update + restart |

**PENTING:** `restart` default delay 2000ms supaya tidak interrupt reply yang sedang dikirim.

---

## Wire Protocol (Summary)

Komunikasi Gateway pakai WebSocket + JSON:

```
Client → Gateway:  { type: "req", id, method, params }
Gateway → Client:  { type: "res", id, ok, payload|error }
Gateway → Client:  { type: "event", event, payload, seq? }
```

### Connection Lifecycle

1. Client connect WebSocket
2. **First frame HARUS** `connect` (dengan auth token kalau diset)
3. Gateway reply `hello-ok` (presence, health, stateVersion, uptimeMs)
4. Siap kirim requests dan receive events

### Events Penting

| Event | Kapan |
|-------|-------|
| `agent` | Agent sedang jalan (stream deltas) |
| `chat` | Chat message activity |
| `presence` | Online/offline status |
| `health` | Health status changes |
| `heartbeat` | Periodic heartbeat |
| `cron` | Cron job events |
| `shutdown` | Gateway mau shutdown |

---

## Multiple Gateways (Advanced)

Untuk isolation/redundancy, bisa jalankan beberapa gateway di satu host. **Tapi kebanyakan setup cuma butuh 1.**

Checklist per instance:
- Unique `gateway.port`
- Unique `OPENCLAW_CONFIG_PATH`
- Unique `OPENCLAW_STATE_DIR`
- Unique `agents.defaults.workspace`

```bash
# Instance A
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001

# Instance B
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

---

## Common Failure Signatures

| Error | Penyebab |
|-------|----------|
| `refusing to bind gateway ... without auth` | Non-loopback bind tanpa token/password |
| `EADDRINUSE` / `another gateway instance` | Port conflict — gateway lain sudah jalan |
| `Gateway start blocked: set gateway.mode=local` | Config set ke remote mode |
| `unauthorized` saat connect | Auth mismatch (token salah) |

---

## Safety Guarantees

- Gateway clients **fail fast** kalau gateway tidak available (no silent fallback)
- Invalid first frame (bukan `connect`) → hard close
- Graceful shutdown emit `shutdown` event sebelum socket ditutup
- Events TIDAK di-replay — client harus refresh state kalau ada gap

---

## Arsitektur Setup Kita (my-vps)

```
Laptop/PC (browser + CLI)
    │
    │ SSH Tunnel (port 18789)
    │
my-vps (VPS Ubuntu)
    │
    │ Docker Compose
    │
    ├── openclaw-gateway container
    │     ├── Gateway (ws://0.0.0.0:18789)
    │     ├── WhatsApp listener (Baileys)
    │     ├── Agent Runtime
    │     │   ├── Huda (main, 40 tools, 9 plugins)
    │     │   ├── Sales Agent
    │     │   ├── PM Agent (12 tools)
    │     │   └── Turath Research (7 tools)
    │     ├── Dashboard web UI
    │     └── HTTP API
    │
    ├── Volumes
    │   ├── ~/.openclaw/ → /home/node/.openclaw (config)
    │   ├── ~/.openclaw/workspace/ → workspace (Huda)
    │   ├── ~/.openclaw/workspace-sales/ → workspace (Sales)
    │   └── ~/repos/ → /home/node/repos:ro (codebase reader)
    │
    └── LLM Provider
        └── OpenRouter → Grok 4.1 Fast / Kimi K2.5
```

---

## Referensi

- Gateway Runbook: https://docs.openclaw.ai/gateway
- Architecture: https://docs.openclaw.ai/concepts/architecture
- Agent Runtime: https://docs.openclaw.ai/concepts/agent
- Agent Loop: https://docs.openclaw.ai/concepts/agent-loop
- System Prompt: https://docs.openclaw.ai/concepts/system-prompt
- Context: https://docs.openclaw.ai/concepts/context
- Streaming: https://docs.openclaw.ai/concepts/streaming
- Retry Policy: https://docs.openclaw.ai/concepts/retry
- Wire Protocol: https://docs.openclaw.ai/gateway/protocol
