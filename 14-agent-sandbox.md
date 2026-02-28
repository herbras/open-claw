# OpenClaw: Agent Sandbox & Security

> Isolasi agent di Docker sandbox, tool policy, workspace access, dan security hardening.

---

## Konsep Dasar

Sandbox adalah **container Docker terpisah** tempat agent jalan. Agent di dalam sandbox tidak bisa akses host langsung — hanya bisa pakai tools yang diizinkan.

| Istilah | Penjelasan |
|---------|-----------|
| **Sandbox** | Docker container terisolasi untuk agent |
| **Sandbox Mode** | Kapan sandbox aktif: `off`, `non-main`, `all` |
| **Sandbox Scope** | Granularity container: `session`, `agent`, `shared` |
| **Tool Policy** | Rules allow/deny tools di dalam sandbox |
| **Workspace Access** | Level akses ke workspace: `none`, `ro`, `rw` |
| **Elevated Mode** | "Escape hatch" — jalankan command di host dari sandbox |

---

## Sandbox Modes

### Kapan Sandbox Aktif

| Mode | Perilaku |
|------|----------|
| `off` | Tidak ada sandbox — semua agent jalan langsung di gateway |
| `non-main` (default) | Sandbox aktif untuk session non-main (grup, cron, webhook) |
| `all` | Semua session di-sandbox, termasuk DM utama |

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main"
      }
    }
  }
}
```

### Per-Agent Override

```json
{
  "agents": {
    "entries": [
      {
        "id": "untrusted",
        "sandbox": {
          "mode": "all"
        }
      },
      {
        "id": "trusted",
        "sandbox": {
          "mode": "off"
        }
      }
    ]
  }
}
```

### GOTCHA: "non-main" dan Session Keys

`non-main` berdasarkan `session.mainKey` (default `"main"`), **BUKAN** agent ID.

- Group sessions → selalu punya key sendiri → **akan di-sandbox**
- Cron sessions → key `cron:*` → **akan di-sandbox**
- DM main session → key `main` → **TIDAK di-sandbox**

Jadi kalau pakai `non-main`, semua grup dan cron otomatis di-sandbox, tapi DM tidak.

---

## Sandbox Scope — Granularity Container

| Scope | Perilaku |
|-------|----------|
| `session` (default) | 1 container per session — paling isolated |
| `agent` | 1 container per agent — sessions share container |
| `shared` | 1 container untuk semua — paling hemat resource |

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "scope": "agent"
      }
    }
  }
}
```

**Trade-off:**
- `session` = paling aman, tapi boros container (setiap session baru bikin container baru)
- `agent` = balance antara isolation dan resource
- `shared` = paling hemat, tapi agent bisa saling lihat file

---

## Docker Sandbox Image

Sandbox pakai Docker image yang sama dengan gateway (default), atau bisa custom:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "image": "openclaw:local"
      }
    }
  }
}
```

### Setup Docker Sandbox di Docker-in-Docker

Kalau gateway sudah jalan di Docker, perlu Docker-in-Docker (DinD) atau mount Docker socket:

```yaml
# docker-compose.yml — tambahkan volume mount
services:
  openclaw-gateway:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

**PENTING:** Mount Docker socket = agent bisa control Docker di host. Ini powerful tapi berbahaya. Hanya lakukan kalau memang butuh sandbox.

---

## Tool Policy di Sandbox

Sandbox bisa punya tool restrictions terpisah dari agent-level restrictions:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "tools": {
          "allow": ["group:fs", "group:web", "message"],
          "deny": ["exec", "process", "browser"]
        }
      }
    }
  }
}
```

### Urutan Filtering Tool

```
Global tools.allow/deny
    ↓
Agent-level tools.allow/deny
    ↓
Sandbox tools.allow/deny
    ↓
Sub-agent tools.allow/deny
```

Setiap level hanya bisa **further restrict**, TIDAK bisa grant back tools yang sudah di-deny.

---

## Workspace Access

Control akses sandbox ke workspace files:

| Level | Perilaku |
|-------|----------|
| `none` | Tidak ada akses ke workspace |
| `ro` (read-only) | Bisa baca, tidak bisa tulis |
| `rw` (read-write) | Full access |

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "workspaceAccess": "ro"
      }
    }
  }
}
```

**Rekomendasi:**
- Agent yang hanya perlu baca SOUL.md → `ro`
- Agent yang perlu tulis memory → `rw`
- Agent untrusted → `none` (inject context lewat system prompt saja)

---

## Security Hardening

### Network Isolation

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "network": "none"
      }
    }
  }
}
```

| Value | Perilaku |
|-------|----------|
| `none` | Tidak ada akses network — paling aman |
| `bridge` | Akses network via Docker bridge |
| (default) | Ikut Docker default |

### Resource Limits

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "pidsLimit": 100,
        "memory": "512m"
      }
    }
  }
}
```

| Key | Default | Penjelasan |
|-----|---------|-----------|
| `pidsLimit` | — | Max jumlah process di container |
| `memory` | — | Max memory (misal `"512m"`, `"1g"`) |

### Seccomp Profile

Restrict system calls yang boleh dipanggil:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "seccomp": "default"
      }
    }
  }
}
```

---

## Multi-Agent Sandbox

Kalau punya beberapa agent dengan level trust berbeda:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main"
      }
    },
    "entries": [
      {
        "id": "main",
        "name": "Huda",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "pm",
        "name": "PM Agent",
        "sandbox": {
          "mode": "all",
          "scope": "agent",
          "tools": {
            "allow": ["group:sessions", "message", "code_tree", "code_grep", "code_read"],
            "deny": ["exec", "process", "browser"]
          },
          "workspaceAccess": "ro"
        }
      },
      {
        "id": "untrusted",
        "name": "Guest Agent",
        "sandbox": {
          "mode": "all",
          "scope": "session",
          "tools": {
            "allow": ["message", "web_search"],
            "deny": ["*"]
          },
          "workspaceAccess": "none",
          "network": "none"
        }
      }
    ]
  }
}
```

### Session Tools Visibility di Sandbox

Sandbox agent punya restricted view ke sessions:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "sessionToolsVisibility": "spawned"
      }
    }
  }
}
```

| Value | Scope |
|-------|-------|
| `spawned` (default) | Hanya session saat ini + spawned sub-agents |
| `all` | Semua sessions (override restriction) |

Kalau `sessionToolsVisibility: "spawned"`, OpenClaw paksa `tools.sessions.visibility` ke `"tree"` meski diset `"all"`.

---

## Elevated Mode — Escape dari Sandbox

Kalau agent di sandbox perlu akses host untuk command tertentu:

```json
{
  "tools": {
    "elevated": {
      "enabled": true
    }
  },
  "agents": {
    "entries": [
      {
        "id": "pm",
        "tools": {
          "elevated": { "enabled": true }
        }
      }
    ]
  }
}
```

**Dua-duanya harus allow:** global `tools.elevated` DAN agent-level `tools.elevated`.

Agent pakai `exec` dengan `elevated: true`:
```json
{
  "command": "git pull",
  "elevated": true
}
```

Ini jalankan command di host (gateway), bukan di sandbox container.

---

## Verifikasi Sandbox

### Cek Agent Resolution

```bash
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node dist/index.js agents list --bindings"
```

### Cek Sandbox Containers

```bash
ssh my-vps "docker ps --filter 'name=openclaw-sbx-'"
```

### Monitor Logs

```bash
ssh my-vps "cd ~/openclaw && docker compose logs -f openclaw-gateway" | grep -E "routing|sandbox|tools"
```

---

## Config Setup Kita (my-vps)

Saat ini kita **TIDAK pakai sandbox** (mode `off` implicit karena tidak diset). Semua agent jalan langsung di gateway container.

Kalau mau enable sandbox nanti:
```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "scope": "agent",
        "workspaceAccess": "rw"
      }
    }
  }
}
```

Ini akan sandbox semua grup dan cron sessions, tapi DM tetap langsung.

---

## Troubleshooting

### Agent tidak ter-sandbox meski mode "all"

1. Cek agent-level override — mungkin ada `sandbox.mode: "off"` di agent config
2. Agent-specific config takes precedence atas defaults

### Tools masih available meski di-deny

1. Urutan filtering: global → agent → sandbox → subagent
2. Setiap level hanya restrict, tidak grant back
3. Cek log: `[tools] filtering tools for agent:<agentId>`

### Container tidak isolated per agent

1. Set `scope: "agent"` — default `"session"` bikin 1 container per session
2. Cek config:
   ```bash
   ssh my-vps "python3 -c \"
   import json
   with open('/home/ubuntu/.openclaw/openclaw.json') as f:
       cfg = json.load(f)
   print(json.dumps(cfg.get('agents',{}).get('defaults',{}).get('sandbox',{}), indent=2))
   \""
   ```

### Docker socket permission error

```bash
ssh my-vps "sudo chmod 666 /var/run/docker.sock"
```

Atau tambahkan user ke group docker:
```bash
ssh my-vps "sudo usermod -aG docker $(whoami)"
```

---

## Referensi

- Sandboxing: https://docs.openclaw.ai/gateway/sandboxing
- Multi-Agent Sandbox: https://docs.openclaw.ai/tools/multi-agent-sandbox-tools
- Elevated Mode: https://docs.openclaw.ai/tools/elevated
- Tool Policy: https://docs.openclaw.ai/tools
