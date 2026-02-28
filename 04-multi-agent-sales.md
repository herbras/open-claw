# 04 - Multi-Agent: Warung Nusantara Sales

Setup agent AI terpisah untuk jualan menu Warung Nusantara di WhatsApp, berdampingan dengan Huda (mentor bisnis) di gateway yang sama.

## Routing

| Source | Agent |
|--------|-------|
| 2 grup WA (Warung Nusantara) | `warung-sales` (Sales) |
| DM WhatsApp | `main` / Huda (Mentor, default) |

## Struktur File di my-vps

```
~/.openclaw/
├── openclaw.json          # Config utama (agents.list + bindings)
├── workspace/             # Workspace Huda (main agent)
│   ├── SOUL.md
│   └── IDENTITY.md
├── workspace-warung-sales/  # Workspace Warung Nusantara
│   ├── SOUL.md            # Persona + menu lengkap + aturan sales
│   └── IDENTITY.md        # Identity sales agent
└── agents/
    ├── main/              # State Huda
    └── warung-sales/        # State Warung Nusantara
        ├── agent/
        └── sessions/
```

## Config Changes (openclaw.json)

Ditambahkan `agents.list` dan `bindings`:

```json
{
  "agents": {
    "defaults": { "..." },
    "list": [
      { "id": "main", "default": true, "name": "Huda", "workspace": "/home/node/.openclaw/workspace" },
      { "id": "warung-sales", "name": "Warung Nusantara Sales", "workspace": "/home/node/.openclaw/workspace-warung-sales" }
    ]
  },
  "bindings": [
    { "agentId": "warung-sales", "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "<GROUP_JID_1>@g.us" } } },
    { "agentId": "warung-sales", "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "<GROUP_JID_2>@g.us" } } }
  ]
}
```

## SOUL.md Warung Nusantara

Berisi:
- Persona sales: ramah, hangat, bahasa santai
- Menu lengkap dengan harga (embed langsung)
- CTA: arahkan ke WA 08xx xxxx xxxx
- Link gambar menu: https://s.id/your-menu-link
- Aturan: jangan bikin harga sendiri, jangan ubah menu

## Verifikasi

```bash
# Cek log setelah restart
ssh my-vps "cd ~/openclaw && docker compose logs --tail=20 openclaw-gateway"

# Cek config di dalam container
ssh my-vps "cd ~/openclaw && docker compose exec openclaw-gateway node -e \"const c=require('/home/node/.openclaw/openclaw.json'); console.log(c.agents?.list?.map(a=>a.id))\""

# Restart gateway
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

## Tanggal Setup

2026-02-07
