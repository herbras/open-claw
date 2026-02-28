# OpenClaw: Broadcast Groups

> Satu pesan masuk, beberapa agent balas — cara setup broadcast groups untuk specialized teams dan multi-perspective replies.
>
> **Status: Experimental** (sejak 2026.1.9). Saat ini hanya WhatsApp. Telegram, Discord, Slack: planned.

---

## Konsep Dasar

Biasanya, satu pesan di grup → satu agent jawab. Dengan **broadcast groups**, satu pesan bisa di-route ke **beberapa agent sekaligus**, masing-masing jawab dari perspektifnya sendiri.

| Istilah | Penjelasan |
|---------|-----------|
| **Broadcast Group** | Grup yang di-handle oleh beberapa agent |
| **Strategy** | Cara agent reply: parallel (default) |
| **Session Isolation** | Setiap agent punya session terpisah meski di grup yang sama |

**PENTING:** Broadcast config adalah **top-level key** di `openclaw.json`, BUKAN nested di `channels.whatsapp.groups`. Broadcast juga **mengambil prioritas di atas bindings** — kalau grup ada di broadcast, bindings diabaikan untuk grup itu.

---

## Kapan Pakai Broadcast Groups

**Use cases:**
- **Multi-language support** — 1 agent jawab bahasa Indonesia, 1 agent jawab bahasa Inggris
- **Specialized teams** — 1 agent sales, 1 agent teknikal, jawab dari angle masing-masing
- **Review/feedback** — Beberapa agent kasih perspektif berbeda ke 1 pertanyaan
- **Mentor + Assistant** — Senior agent kasih jawaban utama, junior agent kasih tambahan

**JANGAN pakai broadcast untuk:**
- Grup biasa dengan 1 bot — pakai routing biasa (lihat `04-multi-agent-sales.md`)
- Kalau hanya butuh sub-agents — pakai `sessions_spawn` (lebih efisien)

---

## Config Setup

### Langkah 1: Definisikan Agents

Pastikan agents sudah didefinisikan di `openclaw.json`:

```json
{
  "agents": {
    "entries": [
      {
        "id": "mentor",
        "name": "Huda",
        "workspace": "/home/node/.openclaw/workspace"
      },
      {
        "id": "sales",
        "name": "Sales Agent",
        "workspace": "/home/node/.openclaw/workspace-sales"
      }
    ]
  }
}
```

### Langkah 2: Setup Broadcast Group

**GOTCHA KRITIS:** Broadcast adalah **top-level key** di `openclaw.json`. BUKAN di dalam `channels.whatsapp.groups`!

```json
// BENAR — top-level broadcast key
{
  "broadcast": {
    "120363403215116621@g.us": ["mentor", "sales"]
  }
}
```

```json
// SALAH — JANGAN nested di channels
{
  "channels": {
    "whatsapp": {
      "groups": {
        "120363403215116621@g.us": {
          "broadcast": { "agents": ["mentor", "sales"] }
        }
      }
    }
  }
}
```

Key-nya adalah peer ID WhatsApp:
- Grup: group JID (misal `120363403215116621@g.us`)
- DM: nomor E.164 (misal `+15555550123`)

Value-nya adalah array agent IDs.

### Langkah 3: Restart Gateway

```bash
ssh my-vps "cd ~/openclaw && docker compose restart openclaw-gateway"
```

---

## Strategy

### Parallel (Default)

Semua agent proses pesan secara bersamaan. Siapa yang selesai duluan, reply duluan.

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["mentor", "sales"]
  }
}
```

`strategy` adalah sibling key sejajar dengan peer ID mappings.

**Pro:** Cepat — semua agent kerja bareng
**Con:** Reply bisa "berantakan" (urutan tidak terprediksi), semua agent count toward rate limits

---

## Session Isolation

Setiap agent di broadcast group punya **session terpisah**:

```
Grup "Project Team" (broadcast: [mentor, sales])
├── mentor session: agent:mentor:whatsapp:group:120363...
└── sales session:  agent:sales:whatsapp:group:120363...
```

Artinya:
- Setiap agent punya history sendiri
- Compaction, memory, dan context terpisah
- Agent A tidak lihat apa yang agent B tulis (kecuali lewat session tools)
- Reset session satu agent tidak affect agent lainnya

---

## Contoh: Multi-Language Support

```json
{
  "agents": {
    "entries": [
      {
        "id": "id-support",
        "name": "Support ID",
        "workspace": "/home/node/.openclaw/workspace-id"
      },
      {
        "id": "en-support",
        "name": "Support EN",
        "workspace": "/home/node/.openclaw/workspace-en"
      }
    ]
  },
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["id-support", "en-support"]
  }
}
```

User kirim pertanyaan → dua agent jawab: satu dalam bahasa Indonesia, satu dalam bahasa Inggris.

---

## Broadcast + Activation Mode

Broadcast groups tetap mengikuti activation mode dan channel allowlists:

- **Mode `mention`**: hanya broadcast kalau pesan @mention bot atau reply ke bot
- **Mode `always`**: semua pesan di-broadcast ke semua agent

Broadcast **TIDAK bypass** channel allowlists atau group activation rules. Dia hanya mengubah _agents mana yang jalan_.

Activation mode tetap diatur lewat `channels.whatsapp.groups`:
```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "120363403215116621@g.us": { "requireMention": false }
      }
    }
  },
  "broadcast": {
    "120363403215116621@g.us": ["mentor", "sales"]
  }
}
```

---

## Best Practices

1. **Batasi jumlah agent** — 2-3 agent per broadcast group cukup. Lebih dari itu bikin spam.

2. **Beda perspektif, bukan duplikat** — Pastikan setiap agent punya angle berbeda. Kalau dua agent jawab hal yang sama, user bingung.

3. **Sequential untuk percakapan** — Kalau agents perlu "saling dengar", pakai sequential supaya agent kedua bisa lihat jawaban agent pertama.

4. **Parallel untuk kecepatan** — Kalau setiap agent independen (multi-language, multi-domain), pakai parallel.

5. **SOUL.md yang jelas** — Di SOUL.md setiap agent, jelaskan "kamu salah satu dari beberapa agent di grup ini". Supaya agent tidak bingung kenapa ada jawaban lain di thread.

---

## Limitations

- **WhatsApp only** — Saat ini hanya didukung untuk WhatsApp. Telegram, Discord, Slack: planned.
- **Experimental** — Fitur ini masih eksperimental, bisa berubah di versi mendatang.
- **Tidak ada koordinasi antar agent** — Agent tidak tahu agent lain sedang proses. Bisa duplikat jawaban.
- **Resource usage 2x+** — Setiap agent pakai LLM call sendiri. Broadcast 3 agent = 3x API cost.
- **Semua agent count toward WhatsApp rate limits** — Hati-hati kalau banyak agent.
- **Max agents** — Tidak ada hard limit, tapi 10+ agent bisa lambat.
- **Memory terpisah** — Agent A tidak otomatis tahu apa yang agent B "ingat". (Shared context mode: planned)
- **Broadcast > Bindings** — Kalau grup ada di `broadcast`, config `bindings` diabaikan untuk grup itu.

---

## Troubleshooting

### Hanya 1 agent yang reply

1. Cek config `broadcast.agents` — semua agent ID harus valid
2. Cek agent aktif: `docker compose logs | grep "agent:"`
3. Cek agent kedua tidak error: `docker compose logs | grep "error"`

### Reply duplikat (agent jawab hal yang sama)

1. Beda-kan SOUL.md — jelaskan role masing-masing
2. Pertimbangkan switch ke sequential (agent kedua lihat jawaban pertama)
3. Tambahkan instruksi di SOUL.md: "Kalau pertanyaan sudah dijawab agent lain, tambahkan perspektif baru saja"

### Broadcast terlalu spam

1. Kurangi jumlah agent
2. Pakai mode `mention` — hanya broadcast kalau di-@mention
3. Set `debounceMs` tinggi supaya pesan cepat digabung

---

## Referensi

- Broadcast Groups: https://docs.openclaw.ai/channels/broadcast-groups
- Multi-Agent Routing: https://docs.openclaw.ai/concepts/multi-agent
