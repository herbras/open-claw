# OpenClaw: Tools & Built-in Tools

> Overview semua built-in tools OpenClaw, cara allow/deny, tool profiles, dan fitur penting seperti Lobster, Elevated Mode, dan Reactions.

---

## Konsep Dasar

Tools adalah **fungsi yang bisa dipanggil agent**. Setiap kali agent perlu "melakukan sesuatu" (cari web, jalankan command, kirim pesan), dia panggil tool.

OpenClaw punya 2 kategori tools:
1. **Built-in tools** — sudah ada dari sananya (exec, browser, message, cron, dll)
2. **Plugin tools** — dari plugin yang kamu install (misal `generate_image`, `turath_search`)

---

## Tool Inventory — Semua Built-in Tools

| Tool | Fungsi | Contoh Pakai |
|------|--------|-------------|
| `exec` | Jalankan shell command di workspace | Build project, git operations |
| `process` | Manage background exec sessions | Poll status, kill, log |
| `apply_patch` | Apply structured patches ke file (OpenAI models only) | Multi-hunk edits |
| `web_search` | Search web via Brave Search API | Riset topik terbaru |
| `web_fetch` | Fetch & extract content dari URL | Baca artikel, docs |
| `browser` | Control browser (OpenClaw-managed) | Scrape JS-heavy sites, screenshot |
| `canvas` | Drive node Canvas (present, eval) | Render UI, A2UI |
| `nodes` | Discover & target paired nodes | Notify, camera, screen record |
| `image` | Analyze gambar dengan image model | Describe screenshot |
| `message` | Kirim pesan ke channel (WA, Telegram, dll) | Reply, react, poll |
| `cron` | Manage cron jobs & wakeups | Schedule tasks |
| `gateway` | Restart/update gateway | Config apply, restart |
| `sessions_list` | List sessions aktif | Lihat semua percakapan |
| `sessions_history` | Lihat transcript session | Debug, inspect |
| `sessions_send` | Kirim pesan ke session lain | Inter-agent communication |
| `sessions_spawn` | Spawn sub-agent | Parallel tasks |
| `session_status` | Cek status session | Current model, context usage |
| `agents_list` | List agents yang boleh di-spawn | Discover available agents |
| `memory_search` | Semantic search dari long-term memory | Recall info lama |
| `memory_get` | Read spesifik file/line dari memory | Targeted recall |

---

## Allow & Deny — Kontrol Tools per Agent

### Global Allow/Deny

Di `openclaw.json`, kamu bisa restrict tools secara global:

```json
{
  "tools": {
    "deny": ["browser", "exec"]
  }
}
```

**Rules:**
- `deny` selalu menang atas `allow`
- Matching case-insensitive
- Wildcard `*` didukung (`"*"` = semua tools)
- Kalau `tools.allow` hanya referensi tools yang unknown/unloaded, OpenClaw log warning dan ignore allowlist (supaya core tools tetap jalan)

### Per-Agent Override

Setiap agent bisa punya allow/deny sendiri:

```json
{
  "agents": {
    "entries": [
      {
        "id": "sales",
        "tools": {
          "allow": ["message", "web_search", "web_fetch"],
          "deny": ["exec", "browser", "apply_patch"]
        }
      }
    ]
  }
}
```

---

## Tool Profiles — Preset Allowlist

Profile adalah **preset** tools yang bisa langsung dipakai. Jadi tidak perlu list satu-satu:

| Profile | Tools yang Tersedia |
|---------|-------------------|
| `minimal` | `session_status` saja |
| `coding` | `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `image` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status` |
| `full` | Semua (sama seperti tidak diset) |

### Contoh: Messaging-only Agent

```json
{
  "tools": {
    "profile": "messaging",
    "allow": ["slack", "discord"]
  }
}
```

### Contoh: Coding Profile tapi Deny Exec

```json
{
  "tools": {
    "profile": "coding",
    "deny": ["group:runtime"]
  }
}
```

### Contoh: Global Coding, Support Agent Messaging-only

```json
{
  "tools": { "profile": "coding" },
  "agents": {
    "entries": [
      {
        "id": "support",
        "tools": { "profile": "messaging", "allow": ["slack"] }
      }
    ]
  }
}
```

---

## Tool Groups — Shorthand

Supaya tidak perlu tulis tools satu-satu, pakai `group:*`:

| Group | Isi |
|-------|-----|
| `group:runtime` | `exec`, `bash`, `process` |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:web` | `web_search`, `web_fetch` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:openclaw` | Semua built-in tools (exclude plugin tools) |

Contoh — allow file tools + browser saja:
```json
{
  "tools": {
    "allow": ["group:fs", "browser"]
  }
}
```

---

## Provider-Specific Tool Policy

Restrict tools untuk provider/model tertentu:

```json
{
  "tools": {
    "profile": "coding",
    "byProvider": {
      "google-antigravity": { "profile": "minimal" },
      "openai/gpt-5.2": { "allow": ["group:fs", "sessions_list"] }
    }
  }
}
```

Ini berguna kalau model tertentu "error-prone" dengan tools tertentu.

---

## Exec Tool — Jalankan Shell Commands

Tool paling powerful (dan paling berbahaya). Jalankan command apapun di workspace.

### Parameters Penting

| Param | Default | Penjelasan |
|-------|---------|-----------|
| `command` | (required) | Command yang dijalankan |
| `yieldMs` | `10000` | Auto-background setelah timeout (ms) |
| `background` | `false` | Langsung background |
| `timeout` | `1800` | Kill process setelah N detik |
| `elevated` | `false` | Jalankan di host (bukan sandbox) |
| `pty` | `false` | Butuh real TTY |
| `host` | — | `sandbox`, `gateway`, atau `node` |

### Security Levels

| Level | Perilaku |
|-------|----------|
| `deny` | Tolak semua exec |
| `allowlist` | Hanya command yang di-whitelist |
| `full` | Semua command diizinkan |

**GOTCHA:** Kalau `process` tool di-deny, `exec` jalan synchronous dan ignore `yieldMs`/`background`.

---

## Lobster — Typed Workflow Runtime

Lobster adalah tool untuk workflow yang lebih complex — resumable approvals, typed steps.

**Require:** Lobster CLI harus tersedia di gateway host.

---

## Elevated Mode — Keluar dari Sandbox

Kalau agent jalan di sandbox tapi perlu akses host:

```json
{
  "tools": {
    "elevated": {
      "enabled": true
    }
  }
}
```

Per-agent override:
```json
{
  "agents": {
    "entries": [
      {
        "id": "main",
        "tools": {
          "elevated": { "enabled": true }
        }
      }
    ]
  }
}
```

**PENTING:** `elevated` = `host=gateway` + `security=full`. Hanya berpengaruh kalau agent sedang di-sandbox. Di luar sandbox, ini no-op.

---

## Reactions — Auto-React Emoji

Bot bisa kasih emoji reaction saat terima pesan (tanda "lagi mikir").

### Format Simple (yang kita pakai)

```json
{
  "messages": {
    "ackReactionScope": "group-mentions"
  }
}
```

| Value | Perilaku |
|-------|----------|
| `"group-mentions"` | React emoji di grup hanya kalau di-@mention |
| `"all"` | React di semua pesan (DM + grup) |
| `"none"` | Tidak react |

### Format Detail (per-channel)

```json
{
  "channels": {
    "whatsapp": {
      "ackReaction": {
        "emoji": "👀",
        "direct": true,
        "group": "mentions"
      }
    }
  }
}
```

| Key | Nilai | Penjelasan |
|-----|-------|-----------|
| `emoji` | `"👀"`, `"✅"`, dll | Emoji yang dipakai |
| `direct` | `true`/`false` | React di DM? |
| `group` | `"always"` / `"mentions"` / `"never"` | Kapan react di grup |

---

## Loop Detection — Anti Loop Tool Call

Kalau agent stuck panggil tool yang sama berulang-ulang:

```json
{
  "tools": {
    "loopDetection": {
      "enabled": true,
      "warningThreshold": 10,
      "criticalThreshold": 20,
      "globalCircuitBreakerThreshold": 30,
      "historySize": 30,
      "detectors": {
        "genericRepeat": true,
        "knownPollNoProgress": true,
        "pingPong": true
      }
    }
  }
}
```

| Detector | Apa yang Dideteksi |
|----------|-------------------|
| `genericRepeat` | Tool + params yang sama berulang |
| `knownPollNoProgress` | Poll tool dengan output identik |
| `pingPong` | Pola bolak-balik A/B/A/B tanpa progress |

Default: `enabled: false`. Aktifkan kalau agent sering loop.

---

## Cara Tools Dikirim ke Agent

Tools diexpose lewat 2 channel:

1. **System prompt text** — deskripsi readable yang agent baca
2. **Tool schema** — structured function definitions yang dikirim ke model API

Kalau tool tidak muncul di salah satu, agent tidak bisa memanggilnya.

---

## Config Setup Kita (my-vps)

```
Tools per agent:
  Huda (main):    40 tools dari 9 plugin (full access)
  Sales Agent:    Default tools (tidak di-restrict)
  PM Agent:       12 tools (code reader + gitlab + fizzy)
  Turath:         7 tools (turath only)

ackReaction:      👀 emoji saat di-mention di grup
```

Agent tools diatur lewat `agents.entries[].tools.allow`:
```json
{
  "agents": {
    "entries": [
      {
        "id": "pm",
        "tools": {
          "allow": ["code_tree", "code_grep", "code_read",
                    "gitlab_list_projects", "gitlab_list_issues",
                    "gitlab_get_issue", "gitlab_create_issue", "gitlab_add_comment",
                    "fizzy_list_users", "fizzy_list_boards",
                    "fizzy_user_cards", "fizzy_user_weekly_report"]
        }
      }
    ]
  }
}
```

---

## Troubleshooting

### Tool tidak muncul untuk agent

1. Cek `agents.entries[].tools.allow` — tool harus di-list explicitly
2. Cek `tools.deny` global — mungkin di-block di level global
3. Cek plugin loaded: `docker compose logs openclaw-gateway | grep "register"`

### "Tool not found" error di log

Plugin belum ter-load. Cek:
```bash
ssh my-vps "cd ~/openclaw && docker compose logs --tail=20 openclaw-gateway" | grep -i "plugin\|register\|error"
```

### Agent spam tool calls (loop)

Aktifkan loop detection:
```json
{ "tools": { "loopDetection": { "enabled": true } } }
```

### Exec timeout — command lama

Naikkan timeout:
```json
{ "tools": { "exec": { "timeout": 3600 } } }
```
Atau jalankan sebagai background: agent set `background: true` di tool call.

---

## Referensi

- Tools Overview: https://docs.openclaw.ai/tools
- Lobster: https://docs.openclaw.ai/tools/lobster
- Exec Tool: https://docs.openclaw.ai/tools/exec
- Elevated Mode: https://docs.openclaw.ai/tools/elevated
- Web Tools: https://docs.openclaw.ai/tools/web
- Reactions: https://docs.openclaw.ai/tools/reactions
