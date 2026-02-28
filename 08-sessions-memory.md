# OpenClaw: Sessions & Memory

> Cara kerja session management, memory, compaction, dan pruning di OpenClaw.
> Ini fondasi penting — setiap pesan yang masuk diproses lewat session system.

---

## Konsep Dasar

Setiap percakapan di OpenClaw punya **session** — wadah yang menyimpan history chat, konteks, dan state. Session menentukan:
- Siapa yang ngobrol sama siapa
- History percakapan yang dikirim ke LLM
- Kapan percakapan di-reset (mulai baru)

| Istilah | Penjelasan |
|---------|-----------|
| **Session** | Satu sesi percakapan (punya ID unik, history, metadata) |
| **Session Key** | Identifier percakapan (misal `agent:main:whatsapp:dm:+6281234`) |
| **Memory** | Long-term memory yang persist antar session (file di workspace) |
| **Compaction** | Rangkuman otomatis kalau context window penuh |
| **Pruning** | Trim tool results lama dari context (in-memory, tidak ubah file) |

---

## Session Keys — Siapa Ngobrol Sama Siapa

OpenClaw bikin session key berdasarkan tipe chat:

| Tipe Chat | Format Session Key | Contoh |
|-----------|-------------------|--------|
| DM (default) | `agent:<agentId>:main` | `agent:main:main` |
| DM (per-peer) | `agent:<agentId>:dm:<peer>` | `agent:main:dm:+6281234` |
| DM (per-channel-peer) | `agent:<agentId>:<channel>:dm:<peer>` | `agent:main:whatsapp:dm:+6281234` |
| Grup | `agent:<agentId>:<channel>:group:<groupId>` | `agent:main:whatsapp:group:120363...` |
| Cron job | `cron:<jobId>` | `cron:daily-report` |
| Webhook | `hook:<uuid>` | `hook:abc123` |

### dmScope — Bagaimana DM Dikelola

Setting `session.dmScope` menentukan bagaimana DM di-track:

| Value | Perilaku |
|-------|----------|
| `main` (default) | Semua DM share 1 session per agent — kontinuitas lintas device/channel |
| `per-peer` | Session terpisah per nomor HP |
| `per-channel-peer` | Session terpisah per channel + nomor HP |
| `per-account-channel-peer` | Paling granular — terpisah per account + channel + peer |

**GOTCHA:** Grup selalu punya session terpisah. `dmScope` hanya berlaku untuk DM.

### Identity Links — Gabungkan Session Lintas Channel

Kalau user yang sama chat dari WhatsApp dan Telegram, bisa digabungkan:

```json
{
  "session": {
    "identityLinks": {
      "alice": ["telegram:123456789", "discord:987654321012345678"]
    }
  }
}
```

Ini bikin semua channel Alice pakai session key yang sama.

---

## Session Lifecycle — Kapan Session Reset

### Daily Reset (Default)

Default: session reset setiap **jam 4 pagi** waktu lokal gateway.

```json
{
  "session": {
    "reset": {
      "mode": "daily",
      "atHour": 4
    }
  }
}
```

Artinya: kalau chat terakhir kemarin jam 11 malam, dan user chat lagi jam 5 pagi, itu session baru (fresh).

### Idle Reset (Opsional)

Tambahkan `idleMinutes` untuk reset kalau idle terlalu lama:

```json
{
  "session": {
    "reset": {
      "mode": "daily",
      "atHour": 4,
      "idleMinutes": 120
    }
  }
}
```

Kalau dua-duanya diset, **mana yang duluan expire** yang trigger reset.

### Per-Type Override

Beda policy untuk DM, grup, dan thread:

```json
{
  "session": {
    "resetByType": {
      "direct": { "mode": "idle", "idleMinutes": 240 },
      "group": { "mode": "idle", "idleMinutes": 120 },
      "thread": { "mode": "daily", "atHour": 4 }
    }
  }
}
```

### Per-Channel Override

Override policy untuk channel tertentu:

```json
{
  "session": {
    "resetByChannel": {
      "discord": { "mode": "idle", "idleMinutes": 10080 }
    }
  }
}
```

### Manual Reset

User bisa kirim `/new` atau `/reset` di chat untuk mulai session baru.

- `/new` juga bisa terima model: `/new grok-4.1` → ganti model + fresh session
- Kalau dikirim tanpa text lain, bot balas greeting "hello"

---

## Memory — Long-term Memory

Memory adalah catatan yang persist antar session. Beda dengan session history yang di-reset, memory bertahan selamanya (sampai dihapus manual).

### Cara Kerja

1. Agent nulis ke file `memory/` di workspace (otomatis, lewat tool)
2. Sebelum tiap reply, agent bisa search memory untuk konteks relevan
3. Memory di-index pakai embedding untuk semantic search

### Config Memory Search

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "sources": ["memory", "sessions"],
        "experimental": { "sessionMemory": true }
      }
    }
  }
}
```

| Key | Nilai | Penjelasan |
|-----|-------|-----------|
| `sources` | `["memory", "sessions"]` | Cari dari memory files dan session history |
| `experimental.sessionMemory` | `true` | Eksperimental — index dan cari dari session JSONL lama |

**GOTCHA:** `sessionMemory` HARUS di-wrap dalam `experimental: {}`. Kalau taruh langsung di `memorySearch`, akan diabaikan.

### Embedding Provider

Memory search butuh embedding model. OpenClaw auto-select berdasarkan urutan:

1. **Local** — `node-llama-cpp` (gratis, default ~0.6 GB model auto-download)
2. **OpenAI** — `text-embedding-3-small`
3. **Gemini** — `gemini-embedding-001`
4. **Voyage**
5. **Mistral**
6. **Disabled** — kalau semua gagal

Remote providers butuh API key. Auto-select coba dari atas, skip kalau tidak available.

Config manual untuk remote embedding:

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "text-embedding-3-small",
        "remote": {
          "baseUrl": "https://api.openai.com/v1/",
          "apiKey": "YOUR_KEY"
        }
      }
    }
  }
}
```

### Memory Files — Dua Jenis

| File | Fungsi | Kapan Dibaca |
|------|--------|-------------|
| `memory/YYYY-MM-DD.md` | Daily log (append-only) | Today + yesterday di-load saat session start |
| `MEMORY.md` | Curated long-term memory | Hanya di-load di main private session (BUKAN grup) |

```
~/.openclaw/workspace/
├── MEMORY.md                  # Curated memory (preferences, keputusan, facts)
├── memory/
│   ├── 2026-03-01.md          # Daily log hari ini
│   ├── 2026-02-28.md          # Daily log kemarin
│   └── projects.md            # Custom memory file (evergreen, tidak di-decay)
```

**GOTCHA:** `MEMORY.md` hanya di-load di main private session, TIDAK di grup. Untuk info yang harus accessible di semua context, pakai daily logs atau custom memory files.

Di Docker:
```bash
ssh my-vps "ls ~/.openclaw/workspace/memory/"
ssh my-vps "cat ~/.openclaw/workspace/MEMORY.md"
```

### Memory Tools

| Tool | Fungsi |
|------|--------|
| `memory_search` | Semantic search over indexed memory chunks |
| `memory_get` | Targeted read file/line range tertentu. Graceful kalau file tidak ada (return kosong, bukan error) |

Kedua tool hanya aktif kalau `memorySearch.enabled` resolve true.

### Hybrid Search (BM25 + Vector)

Default memory search pakai vector similarity (semantic). Bisa digabung dengan BM25 keyword search untuk hasil lebih baik — terutama untuk exact match (ID, env vars, code symbols):

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "query": {
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "mmr": { "enabled": true, "lambda": 0.7 },
            "temporalDecay": { "enabled": true, "halfLifeDays": 30 }
          }
        }
      }
    }
  }
}
```

| Feature | Default | Penjelasan |
|---------|---------|-----------|
| `hybrid.enabled` | `true` | Gabungkan vector + BM25 |
| `vectorWeight` | `0.7` | Bobot semantic similarity |
| `textWeight` | `0.3` | Bobot keyword match |
| `mmr.enabled` | off | Diversity re-ranking (kurangi duplikat) |
| `temporalDecay` | off | Recency boost — info baru lebih tinggi score-nya |

**Temporal decay:** Dengan halfLife 30 hari: hari ini=100%, 7 hari=84%, 30 hari=50%, 90 hari=12.5%. File evergreen (`MEMORY.md`, non-dated files) TIDAK di-decay.

---

## Compaction — Rangkum Kalau Context Penuh

Setiap LLM punya batas context window (max token yang bisa dilihat). Chat panjang bisa melebihi batas ini. Compaction **merangkum history lama** jadi summary ringkas supaya muat.

### Cara Kerja

1. Session mendekati batas context window
2. OpenClaw trigger auto-compaction
3. Pesan-pesan lama dirangkum jadi 1 summary entry
4. Pesan-pesan baru tetap utuh
5. Summary + pesan baru = context yang dikirim ke LLM

### Config Compaction

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard"
      }
    }
  }
}
```

**Mode `safeguard`** (default): auto-compact kalau context kepanjangan.

### Memory Flush Sebelum Compaction

Sebelum compact, OpenClaw bisa jalankan **silent memory flush** — minta agent simpan catatan penting ke memory sebelum history dirangkum.

**Simple config:**
```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "memoryFlush": true
      }
    }
  }
}
```

**Detailed config (advanced):**
```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 20000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

- **Soft threshold**: flush trigger saat token estimate mendekati `contextWindow - reserveTokensFloor - softThresholdTokens`
- Silent by default (pakai `NO_REPLY`)
- Satu flush per compaction cycle
- Skip kalau workspace read-only

**PENTING:** `memoryFlush: true` sangat direkomendasikan. Tanpa ini, informasi dari chat lama bisa hilang saat compaction.

### Manual Compaction

Kirim `/compact` di chat untuk force compaction:

```
/compact
/compact Fokus ke keputusan dan pertanyaan terbuka
```

Berguna kalau session terasa "berat" atau konteks mulai stale.

---

## Session Pruning — Trim Tool Results

Pruning berbeda dari compaction:
- **Compaction** = rangkum dan simpan ke file (persistent)
- **Pruning** = trim tool results di memory saja (per-request, tidak ubah file)

### Kapan Pruning Jalan

Pruning aktif untuk **Anthropic API** (termasuk OpenRouter Anthropic models):
- Mode `cache-ttl`: prune kalau cache TTL sudah expired
- Hanya trim `toolResult` messages — user dan assistant messages TIDAK diubah

### Config Pruning

```json
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "5m"
      }
    }
  }
}
```

Default values kalau enabled:

| Key | Default | Penjelasan |
|-----|---------|-----------|
| `ttl` | `5m` | Cache TTL sebelum prune |
| `keepLastAssistants` | `3` | Protect N assistant messages terakhir |
| `softTrimRatio` | `0.3` | Ratio untuk soft trim |
| `hardClearRatio` | `0.5` | Ratio untuk hard clear |
| `minPrunableToolChars` | `50000` | Min ukuran tool result sebelum eligible prune |

### Soft vs Hard Pruning

- **Soft trim** — Keep head + tail, sisipkan `...` di tengah. Tool result yang terlalu besar dipotong.
- **Hard clear** — Ganti seluruh tool result dengan placeholder: `"[Old tool result content cleared]"`

**GOTCHA:** Tool results yang berisi **image blocks** TIDAK pernah di-prune.

---

## Session Tools — Inspect & Kirim ke Session Lain

Agent punya built-in tools untuk manage sessions:

| Tool | Fungsi |
|------|--------|
| `sessions_list` | List semua session aktif |
| `sessions_history` | Lihat transcript satu session |
| `sessions_send` | Kirim pesan ke session lain |
| `sessions_spawn` | Spawn sub-agent di session terpisah |
| `session_status` | Cek status session saat ini |

### Visibility (Keamanan)

`tools.sessions.visibility` mengatur session mana yang bisa diakses:

| Value | Scope |
|-------|-------|
| `self` | Hanya session saat ini |
| `tree` (default) | Session saat ini + spawned sub-agents |
| `agent` | Semua session milik agent ini |
| `all` | Semua session (cross-agent) |

```json
{
  "tools": {
    "sessions": {
      "visibility": "tree"
    }
  }
}
```

---

## Config Setup Kita (my-vps)

Ini setting yang sudah jalan di server kita:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "memoryFlush": true
      },
      "memorySearch": {
        "sources": ["memory", "sessions"],
        "experimental": { "sessionMemory": true }
      }
    }
  }
}
```

Artinya:
- Compaction otomatis kalau context kepanjangan
- Sebelum compact, agent simpan catatan ke memory dulu
- Memory search cari dari memory files + session history lama

**GOTCHA:** `sessionMemory` harus di-nested dalam `experimental`, BUKAN langsung di `memorySearch`.

---

## Troubleshooting

### Session tidak reset meski sudah lewat jam 4 pagi

Cek apakah ada override `resetByType` atau `resetByChannel` yang pakai mode `idle`:
```bash
ssh my-vps "python3 -c \"
import json
with open('/home/ubuntu/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
print(json.dumps(cfg.get('session', {}), indent=2))
\""
```

### Memory search tidak return hasil

1. Cek apakah ada file di `memory/`:
   ```bash
   ssh my-vps "ls -la ~/.openclaw/workspace/memory/"
   ```
2. Cek embedding provider — kalau local `node-llama-cpp` gagal, set remote fallback:
   ```json
   {
     "agents": { "defaults": { "memorySearch": { "fallback": "openai" } } }
   }
   ```

### Context terlalu berat / response lambat

1. Cek ukuran session: kirim `/status` di chat
2. Force compaction: kirim `/compact`
3. Mulai fresh: kirim `/new`

### "Context window exceeded" error

Auto-compaction harusnya handle ini, tapi kalau masih error:
- Pastikan `compaction.mode` bukan `"off"`
- Kurangi `bootstrapMaxChars` kalau SOUL.md terlalu panjang
- Kirim `/compact` manual

---

## Command Chat untuk Session

| Command | Fungsi |
|---------|--------|
| `/new` | Mulai session baru (opsional: `/new grok-4.1` ganti model) |
| `/reset` | Reset session (sama seperti `/new`) |
| `/status` | Lihat info session saat ini (context usage, model, dll) |
| `/compact` | Force compaction (opsional: tambah instruksi) |
| `/context list` | Lihat isi system prompt dan workspace files |
| `/context detail` | Detail context contributors terbesar |
| `/stop` | Abort run saat ini + clear queue |
| `/send on` | Allow sending di session ini |
| `/send off` | Block sending di session ini |

---

## GOTCHA & Tips

1. **Session key grup selalu terpisah** — `dmScope` TIDAK berlaku untuk grup. Setiap grup punya session sendiri.

2. **Daily reset jam 4 pagi waktu gateway** — bukan waktu user. Kalau server di timezone berbeda, sesuaikan `atHour`.

3. **Compaction persistent, pruning tidak** — Compaction ubah file JSONL di disk. Pruning hanya trim di memory per-request.

4. **`memoryFlush: true` sangat penting** — Tanpa ini, info dari chat panjang bisa hilang total setelah compaction.

5. **Cron jobs selalu fresh session** — Setiap cron run bikin `sessionId` baru. Tidak ada reuse dari run sebelumnya.

6. **JSONL transcript bisa dibaca langsung** — File ada di `~/.openclaw/agents/{agentId}/sessions/`. Bisa di-inspect manual untuk debug.

---

## Referensi

- Session Management: https://docs.openclaw.ai/concepts/session
- Sessions: https://docs.openclaw.ai/concepts/sessions
- Memory: https://docs.openclaw.ai/concepts/memory
- Compaction: https://docs.openclaw.ai/concepts/compaction
- Session Pruning: https://docs.openclaw.ai/concepts/session-pruning
- Session Tools: https://docs.openclaw.ai/concepts/session-tool
