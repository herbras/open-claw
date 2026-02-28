# OpenClaw: Hooks & Cron Jobs

> Automasi di OpenClaw — hooks untuk react ke event, cron untuk scheduled tasks, dan webhooks untuk trigger dari luar.

---

## Konsep Dasar

| Istilah | Penjelasan |
|---------|-----------|
| **Hook** | Script/command yang jalan otomatis saat event tertentu terjadi |
| **Cron Job** | Task terjadwal (pakai cron expression) |
| **Heartbeat** | Periodic "pulse" yang menjalankan HEARTBEAT.md |
| **Webhook** | HTTP endpoint untuk trigger agent dari luar |

---

## Hooks — React ke Event

Hooks adalah **event-driven scripts**. Setiap kali event tertentu terjadi (misal: boot, command, session start), hook yang terdaftar akan dijalankan.

### Bundled Hooks (Bawaan)

OpenClaw punya 4 bundled hooks:

| Hook | Event | Fungsi |
|------|-------|--------|
| `session-memory` | `command:new` | Simpan session context ke `workspace/memory/` saat `/new` |
| `bootstrap-extra-files` | `agent:bootstrap` | Inject extra files ke bootstrap context via glob patterns |
| `command-logger` | `command` | Log semua command ke `~/.openclaw/logs/commands.log` (JSONL) |
| `boot-md` | `gateway:startup` | Jalankan `BOOT.md` dari workspace saat gateway start |

### Hook Discovery

Hooks di-discover dari 3 direktori (urutan precedence):

1. **Workspace hooks**: `<workspace>/hooks/` — per-agent, highest precedence
2. **Managed hooks**: `~/.openclaw/hooks/` — user-installed, shared
3. **Bundled hooks**: `<install>/dist/hooks/bundled/` — bawaan OpenClaw

Setiap hook adalah folder berisi:
```
my-hook/
  HOOK.md        # Metadata (YAML frontmatter) + dokumentasi
  handler.ts     # Handler implementation
```

### Config Hooks

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false },
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

### Hook Events yang Tersedia

| Event | Kapan Terjadi |
|-------|--------------|
| `command:new` | Saat command `/new` dijalankan |
| `command:reset` | Saat command `/reset` dijalankan |
| `command:stop` | Saat command `/stop` dijalankan |
| `agent:bootstrap` | Saat build bootstrap files sebelum system prompt |
| `gateway:startup` | Saat gateway mulai (setelah channels start + hooks loaded) |
| `message:received` | Pesan masuk diterima |
| `message:sent` | Setelah pesan berhasil dikirim |

**GOTCHA:** Event pakai format `type:action` (colon separator), bukan underscore. Misal `command:new`, BUKAN `command_new`.

### HOOK.md Format

```yaml
---
name: my-hook
description: "Deskripsi singkat"
metadata:
  openclaw:
    emoji: link
    events: ["command:new"]
    requires:
      bins: ["node"]
---
# My Hook

Dokumentasi detail di sini...
```

### Custom Hooks — Handler

```ts
// handler.ts
const myHandler = async (event) => {
  if (event.type !== "command" || event.action !== "new") return;
  console.log(`[my-hook] New command triggered`);
  event.messages.push("My hook executed!");
};
export default myHandler;
```

### Plugin Hook Points (Lengkap)

| Hook | Kapan | Apa yang Bisa Dilakukan |
|------|-------|------------------------|
| `before_model_resolve` | Pre-session, belum ada `messages` | Override provider/model |
| `before_prompt_build` | Setelah session load, ada `messages` | Inject `prependContext`/`systemPrompt` |
| `before_agent_start` | Legacy — bisa jalan di salah satu phase | Prefer 2 hook di atas |
| `agent_end` | Setelah agent selesai | Inspect final messages + metadata |
| `before_tool_call` | Sebelum tool dipanggil | Modify params, block call |
| `after_tool_call` | Setelah tool selesai | Modify results |
| `tool_result_persist` | Sebelum tool result ditulis ke transcript | Transform results |

---

## Cron Jobs — Scheduled Tasks

Cron adalah scheduler bawaan Gateway. Jobs persist di `~/.openclaw/cron/jobs.json` — restart tidak hilangkan jadwal.

### Dua Execution Styles

| Style | Session | Payload | Use Case |
|-------|---------|---------|----------|
| **Main session** | Pakai main session | `systemEvent` | Reminder, nudge |
| **Isolated** | Fresh session `cron:<name>` | `agentTurn` | Report, task mandiri |

### Wake Modes

| Mode | Perilaku |
|------|----------|
| `now` (default) | Execute immediately — trigger heartbeat langsung |
| `next-heartbeat` | Tunggu heartbeat cycle berikutnya |

### Delivery Modes (untuk Isolated jobs)

| Mode | Perilaku |
|------|----------|
| `announce` (default) | Kirim result ke chat channel |
| `webhook` | POST result ke URL |
| `none` | Tidak kirim apapun |

### Quick Start — CLI

```bash
# One-shot reminder (main session)
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js cron add \
  --name 'Reminder' \
  --at '2026-03-01T16:00:00Z' \
  --session main \
  --system-event 'Reminder: cek docs draft' \
  --wake now \
  --delete-after-run"

# Recurring isolated job with delivery
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js cron add \
  --name 'Morning brief' \
  --cron '0 7 * * *' \
  --tz 'Asia/Jakarta' \
  --session isolated \
  --message 'Ringkasan update semalam.' \
  --announce \
  --channel whatsapp \
  --to '+628xxxxxxxxx1'"
```

### Setup via Agent Tool Call

Agent punya `cron` tool yang bisa add/update/remove jobs:

```json
{
  "name": "Morning brief",
  "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "Asia/Jakarta" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Ringkasan update semalam."
  },
  "delivery": {
    "mode": "announce",
    "channel": "whatsapp",
    "to": "+628xxxxxxxxx1",
    "bestEffort": true
  }
}
```

### Config Cron

```json
{
  "cron": {
    "enabled": true,
    "store": "~/.openclaw/cron/jobs.json",
    "maxConcurrentRuns": 1
  }
}
```

Disable: `cron.enabled: false` atau env `OPENCLAW_SKIP_CRON=1`.

### Schedule Kinds

| Kind | Format | Contoh |
|------|--------|--------|
| `at` | ISO 8601 timestamp | `"2026-03-01T16:00:00Z"` |
| `every` | Interval (ms) | `3600000` (setiap 1 jam) |
| `cron` | 5/6-field cron expression | `"0 8 * * *"` (jam 8 pagi) |

**GOTCHA — Stagger:** Untuk recurring top-of-hour cron jobs, OpenClaw apply stagger acak sampai 5 menit supaya tidak thundering herd. Override dengan `schedule.staggerMs` atau `--exact` flag.

### Cron Expression Cheat Sheet

| Expression | Artinya |
|-----------|---------|
| `0 8 * * *` | Setiap hari jam 8 pagi |
| `0 */6 * * *` | Setiap 6 jam |
| `30 9 * * 1-5` | Senin-Jumat jam 9:30 |
| `0 0 1 * *` | Tanggal 1 setiap bulan, jam 00:00 |
| `*/15 * * * *` | Setiap 15 menit |

### Manage Cron lewat Chat

Agent punya `cron` tool bawaan:

| Action | Fungsi |
|--------|--------|
| `cron status` | Lihat status cron system |
| `cron list` | List semua jobs |
| `cron add` | Tambah job baru |
| `cron update` | Update job existing |
| `cron remove` | Hapus job |
| `cron run` | Force run job sekarang |
| `cron runs` | Lihat history runs |
| `cron wake` | Enqueue system event + optional heartbeat |

### Manage Cron lewat CLI

```bash
# List jobs
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js cron list"

# Run job manual
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js cron run daily-report"
```

---

## Cron vs Heartbeat

Dua-duanya periodic, tapi beda:

| | Cron Job | Heartbeat |
|---|---------|-----------|
| **Trigger** | Cron expression (specific time) | Interval-based (misal setiap 1 jam) |
| **Task** | Custom prompt per job | Baca `HEARTBEAT.md` dari workspace |
| **Session** | Fresh session per run | Bisa reuse session |
| **Config** | `cron.jobs[]` | `agents.defaults.heartbeat` |
| **Use case** | Scheduled reports, daily tasks | Keep-alive, periodic checks |

### Config Heartbeat

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "enabled": true,
        "intervalMinutes": 60
      }
    }
  }
}
```

Heartbeat baca file `HEARTBEAT.md` dari workspace dan jalankan instruksi di dalamnya.

**GOTCHA:** Heartbeat dan cron jobs bisa overlap. Gunakan cron untuk task spesifik (jadwal pasti), heartbeat untuk "keep alive" general.

---

## Webhooks — Trigger dari Luar

Webhook bikin HTTP endpoint yang bisa di-hit dari luar untuk trigger agent.

### Konsep

1. OpenClaw expose endpoint di gateway
2. External service (GitHub, Stripe, dll) POST ke endpoint
3. Agent menerima payload dan proses

### Config Webhook

```json
{
  "webhooks": {
    "endpoints": [
      {
        "id": "github-push",
        "path": "/webhook/github",
        "agentId": "pm",
        "secret": "your-webhook-secret",
        "task": "Ada push baru ke repo. Review commit dan buat ringkasan."
      }
    ]
  }
}
```

### Session Key untuk Webhook

Webhook default pakai session key `hook:<uuid>`. Kalau mau custom:

```json
{
  "webhooks": {
    "endpoints": [
      {
        "id": "github-push",
        "sessionKey": "webhook-github"
      }
    ]
  }
}
```

### Test Webhook Manual

```bash
# Hit webhook endpoint
curl -X POST http://localhost:18789/webhook/github \
  -H "Content-Type: application/json" \
  -d '{"ref": "refs/heads/main", "commits": [{"message": "fix: bug login"}]}'
```

**PENTING:** Webhook endpoint accessible dari luar hanya kalau gateway di-expose (bind `lan` + port open). Untuk security, pakai SSH tunnel atau Tailscale.

---

## Automation Troubleshooting

### Cron job tidak jalan

1. Cek job enabled:
   ```bash
   ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js cron list"
   ```
2. Cek timezone gateway — cron pakai timezone host
3. Cek log saat jadwal:
   ```bash
   ssh my-vps "cd ~/openclaw && docker compose logs --tail=50 openclaw-gateway" | grep -i "cron"
   ```

### Hook tidak ter-trigger

1. Cek event name benar (case-sensitive)
2. Cek plugin loaded: `docker compose logs | grep "plugin"`
3. Hook error tidak crash gateway — cek log warning

### Webhook return 404

1. Cek path di config: harus match persis
2. Cek gateway running: `docker compose ps`
3. Cek port accessible: `curl http://localhost:18789/health`

---

## Referensi

- Hooks: https://docs.openclaw.ai/automation/hooks
- Cron Jobs: https://docs.openclaw.ai/automation/cron-jobs
- Cron vs Heartbeat: https://docs.openclaw.ai/automation/cron-vs-heartbeat
- Webhooks: https://docs.openclaw.ai/automation/webhook
- Automation Troubleshooting: https://docs.openclaw.ai/automation/troubleshooting
